# Nouns Descriptor V2

## Simple Summary

A new descriptor version that allows for significantly cheaper deployment of the Nouns art and better composability by improving the external API.

## Abstract

This specification defines a new version of the `NounsDescriptor` contract. The new contracts provides the following benefits:

- Cheaper deployment cost: ~13.8M gas instead of ~67.3M gas
- Easier composability for external contracts by splitting the tokenURI construction to several contracts

## Technical Specification

### Overview

A combination of several techniques will be used in order to reduce the storage size and cost required for the art:

1. RLE encoding supporting multilines
2. Compressing the data using DEFLATE
3. Storing data using `SSTORE2`

#### RLE encoding supporting multilines

The current implementation of the RLE encoding is limited to the length of a row, i.e if it is encoding 2 rows of blue pixels, the encoding will have one RLE per row.
By allowing the encoding to span across several rows, the encoding can be more effecient.

This will require:

1. Updating the encoder (javascript code) to allow encoding multiple lines
2. Updating the decoded (smart contract code) to support multi-line encoding. Currently this is in `MultiPartRLEToSVG.sol`

#### Compressing the data using DEFLATE

1. Compression

   The RLE encoded images can be further compressed by using the general purpose [DEFLATE](https://en.wikipedia.org/wiki/Deflate) compression algorithm.

   In order to maximize the compression effeciency, the images will be compressed in batch. Specifically, all images of a certain trait type (e.g. heads) will be compressed together.

   In each batch, the images will be abi-encoded as `bytes[]` prior to the compression.

   Example javascript code of abi-encoding & compression using deflate:

   ```js
   import { deflateRawSync } from "zlib";

   // data is an array of hexlified byte strings, each starting with 0x
   const abiEncoded = ethers.utils.defaultAbiCoder.encode(["bytes[]"], [data]);
   const encodedCompressed = `0x${deflateRawSync(
     Buffer.from(abiEncoded.substring(2), "hex")
   ).toString("hex")}`;
   ```

2. Decompression

   To decompress the data, a solidity library implementing the [Puff](https://github.com/madler/zlib/tree/master/contrib/puff) decompression algorithm will be used. It will be a slightly modified version of [inflate-sol](https://github.com/adlerjohn/inflate-sol) that introduces gas saving modifications.

#### Storing data using `SSTORE2`

Use the [SSTORE2](https://github.com/Rari-Capital/solmate/blob/main/src/utils/SSTORE2.sol) library for writing and reading the images data. This reduces the cost of storage.

#### Contracts changes

1. `INounsDescriptor`

   1.1. `palettes`

   ```diff
   -    function palettes(uint8 paletteIndex, uint256 colorIndex) external view returns (string memory);
   -    function addColorToPalette(uint8 paletteIndex, string calldata color) external;


   +    /// @return Encoded color palette: every 3 bytes represent an RGB color, e.g 0x112233 = #112233
   +    function palettes(uint8 paletteIndex) external view returns (bytes memory);

   +    function setPalette(uint8 paletteIndex, bytes calldata palette) external;
   ```

   1.2. Change functions for adding new images, shown here only for the body trait, but applies for heads, glasses & accessories:

   ```diff
   -    function addManyBodies(bytes[] calldata bodies) external;
   -    function addBody(bytes calldata body) external;

   +    /**
   +     * @notice Add a batch of body images.
   +     * @param encodedCompressed bytes created by taking a string array of RLE-encoded images, abi encoding it as a bytes array,
   +     * and finally compressing it using deflate.
   +     * @param decompressedLength the size in bytes the images bytes were prior to compression; required input for Inflate.
   +     * @param imageCount the number of images in this batch; used when searching for images among batches.
   +     */
   +    function addBodies(
   +        bytes calldata encodedCompressed,
   +        uint80 decompressedLength,
   +        uint16 imageCount
   +    ) external;

   +    /**
   +     * @notice Add a batch of body images from an existing storage contract.
   +     * @param pointer the address of a contract where the image batch was stored using SSTORE2. The data
   +     * format is expected to be like {encodedCompressed}: bytes created by taking a string array of
   +     * RLE-encoded images, abi encoding it as a bytes array, and finally compressing it using deflate.
   +     * @param decompressedLength the size in bytes the images bytes were prior to compression; required input for Inflate.
   +     * @param imageCount the number of images in this batch; used when searching for images among batches.
   +     */
   +    function addBodiesFromPointer(
   +        address pointer,
   +        uint80 decompressedLength,
   +        uint16 imageCount
   +    ) external;
   ```

   All functions that add images will have access control restriction to the contract owner.

2. `NounsDescriptor`

   The `constructor` will get addresses for 2 contracts, one implementing `INounsArt` and one implementing `ISVGRenderer`.

3. `INounsArt`

   ```js
      function addManyBackgrounds(string[] calldata _backgrounds) external;
      function addBackground(string calldata _background) external;
      function palettes(uint8 paletteIndex) external view returns (bytes memory);
      function setPalette(uint8 paletteIndex, bytes calldata palette) external;

      function addBodies(
          bytes calldata encodedCompressed,
          uint80 decompressedLength,
          uint16 imageCount
      ) external;

      function addBodiesFromPointer(
        address pointer,
        uint80 decompressedLength,
        uint16 imageCount
    ) external;

    /// @notice A batch of compressed images
    struct NounArtStoragePage {
        /// @dev Number of images in this page
        uint16 imageCount;
        /// @dev Length of decompressed data, needed for decompression
        uint80 decompressedLength;
        /// @dev Pointer to the stored data to be read with SSTORE2
        address pointer;
    }

    struct Trait {
        /// @notice Array of pages, each holding a batch of images
        NounArtStoragePage[] storagePages;
        /// @notice Total stored images across all pages
        uint256 storedImagesCount;
    }

    /// Same functions below for other traits (heads, glasses, accessories)

    /// @notice returns an RLE encoded image of body trait
    function bodies(uint256 storageIndex) external view returns (bytes memory);

    /// @notice Returns a Trait object for the body trait images
    function bodiesTrait() external view returns (Trait memory);

    /// @notice Returns the number of pages in the body trait Trait object
    function bodiesPageCount() external view returns (uint256);

    /// @notice Returns page number `pageIndex` of the body trait Trait object
    function bodiesPage(uint256 pageIndex) external view returns (INounsArt.NounArtStoragePage memory);

   ```

   The `NounsArt` contract will be responsible for storing and retrieving the images.
   Changes to the images will be access controlled and restricted to the active `NounsDescriptor`.

   Each time a batch of images is added for a certain trait, a new "page" is created and represented by `NounArtStoragePage`. This page is then added to the `storedPages` member of the `Trait` struct for the "body" trait.
   Same behavior for the other traits (heads, glasses, accessories).

4. `NFTDescriptor`

   - The public functions now receive an `ISVGRenderer` as a parameter.
   - The palette is now passed as part of the `TokenURIParams`.

   ```diff
   -    function constructTokenURI(TokenURIParams memory params, mapping(uint8 => string[]) storage palettes) public view returns (string memory)
   +    function constructTokenURI(ISVGRenderer renderer, TokenURIParams memory params) public view returns (string memory)

   -    function generateSVGImage(MultiPartRLEToSVG.SVGParams memory params, mapping(uint8 => string[]) storage palettes) public view returns (string memory)
   +    function generateSVGImage(ISVGRenderer renderer, ISVGRenderer.SVGParams memory params) public view returns (string memory)
   ```

5. `ISVGRenderer`

   `SVGRenderer` is responsible for taking RLE encoded image parts and constructing an SVG.
   This is similar to the code currently in the `MultiPartRLEToSVG.sol` library.

   ```js
   struct Part {
       /// @dev RLE encoded image
       bytes image;
       bytes palette;
   }

   struct SVGParams {
       Part[] parts;
       string background;
   }

   function generateSVG(SVGParams memory params) external view returns (string memory svg);
   function generateSVGPart(Part memory part) external view returns (string memory partialSVG);
   function generateSVGParts(Part[] memory parts) external view returns (string memory partialSVG);
   ```

### Implementation

Not started.
