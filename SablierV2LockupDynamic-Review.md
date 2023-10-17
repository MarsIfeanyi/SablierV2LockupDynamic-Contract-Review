# The Sablier Protocol

Sablier is a token streaming protocol developed with Ethereum
smart contracts, designed to facilitate by-the-second payments for cryptocurrencies, specifically ERC-20
assets. The protocol employs a set of persistent and non-upgradable smart contracts that prioritize security, censorship resistance, self-custody, and functionality without the need for trusted intermediaries who may selectively restrict access.

- The Sablier protocols uses ERC-1620: Money Streaming as specified in the [EIP-1620](https://eips.ethereum.org/EIPS/eip-1620), proposed by Paul Berg the co-founder of Sablier
  Money streaming represents the idea of continuous payments over a finite period of time. Block numbers are used as a proxy of time to continuously update balances.

- As an immutable protocol, Sablier is not upgradeable, meaning that no party can pause the contracts, reverse transactions, or alter the users' streams in any way. This ensures the system remains transparent, secure, and resistant to manipulation or abuse.

- Sablier introduces the concept of asset streaming, enabling users to make continuous, real-time payments on a per-second basis. This innovative approach enables seamless, frictionless transactions and promotes increased financial flexibility for users, businesses, and other entities. Sablier makes the passage of time itself the trust-binding mechanism, unlocking business opportunities that were previously unavailable.

- Currently, there are two major versions of the protocol:
  - V1, which is available under GNU V3, and
  - V2, which is licensed under BUSL-1.1.

## Streaming

Asset streaming means the ability to make continuous, real-time payments on a per-second basis. This novel approach to making payments is the core concept of Sablier.

## Types of Streams

- LockUp stream
- LockUp Linear stream
- Cliff
- LockUp Dynamic streams
- Exponential
- Exponential Cliff
- Unlock in Steps

## Contract Overview

### LockUp Dynamic Stream:

A Lockup Dynamic stream is a type of stream in the Sablier protocol where the payment rate per second can vary over time. it has

- Segment: The Segment helps facilitate the calculation of the custom streaming curve, which determines how the payment rate varies over time.
- Milestone: The milestone is like the time marker. It represents a point in time when the payment rate changes according to the defined streaming curve.

Envision an hourglass, with grains of sand steadily flowing through it. Now, replace the sand with your crypto assets and the hourglass with Sablier. There you have it: a clear understanding of token streaming.

As an example, suppose you stream $100 worth of tokens to Bob over a month. You would first deposit the $100 in Sablier, and then, every second, Bob will receive a fraction of those tokens. Bob will be earning tokens in real time. At the end of the month, Bob will have received all funds. But Bob can already withdraw the funds that have already been streamed during the month.

## Table of Contents

- [The Sablier Protocol](#the-sablier-protocol)
  - [Streaming](#streaming)
  - [Types of Streams](#types-of-streams)
  - [Contract Overview](#contract-overview)
    - [LockUp Dynamic Stream:](#lockup-dynamic-stream)
  - [Table of Contents](#table-of-contents)
  - [Review Scope:](#review-scope)
    - [Libraries](#libraries)
    - [Storage](#storage)
    - [Constructor Function:](#constructor-function)
      - [Construction function parameters:](#construction-function-parameters)
    - [Public View Getter Functions](#public-view-getter-functions)
    - [External functions](#external-functions)
    - [Internal View functions](#internal-view-functions)

## Review Scope:

This review covers only the SablierV2LockUpDynamic contract located inside the `src/SablierV2LockUpDynamic.sol`

A Lockup Dynamic stream can be composed of multiple segments, which are separate partitions with different streaming amount and rates.
The protocol uses these segments to enable custom streaming curves, which power exponential streams, cliff streams, etc.

### Libraries

`SablierV2LockupDynamic.sol` uses the following libraries defined using the `A for B` pattern:

```sh
using CastingUint128 for uint128;
 using CastingUint40 for uint40;
 using SafeERC20 for IERC20;
```

### Storage

`SablierV2LockupDynamic.sol` has one mapping called `_streams`, it maps the `StreamId` to `Stream` struct defined in the `LockupDynamic` library in the `src/types/DataTypes.sol`.

- The streamId is generated each time a new stream is created and can be queried to get any detail about the Stream as defined in the stream Struct. The streamId is the key, which can be queried to all details or data in the `Stream` struct.

```sh
mapping(uint256 id => LockupDynamic.Stream stream) private _streams;
```

The `Stream` struct has tight variable packing to save gas.

```sh
 struct Stream {
    // slot 0
    address sender;
    uint40 startTime;
    uint40 endTime;
    bool isCancelable;
    bool wasCanceled;
    // slot 1
    IERC20 asset;
    bool isDepleted;
    bool isStream;
    bool isTransferable;
    // slot 2 and 3
    Lockup.Amounts amounts;
    // slots [4..n]
    Segment[] segments;
  }

```

Inside the Stream Struct has two structs, `Segments` and `Amounts`

`Segment` is a struct with three fields:

```sh
struct Segment {
    uint128 amount;
    UD2x18 exponent;
    uint40 milestone;
  }
```

`Amounts` is a struct with three fields, defined in the `Lockup` library inside the `src/types/DataTypes.sol`

```sh
  struct Amounts {
   // slot 0
   uint128 deposited;
   uint128 withdrawn;
   // slot 1
   uint128 refunded;
 }

```

### Constructor Function:

The constructor of `SablierV2LockUpDynamic` accepts the following parameter and initializes them upon the contract deployment.

```sh
constructor(
    address initialAdmin,
    ISablierV2Comptroller initialComptroller,
    ISablierV2NFTDescriptor initialNFTDescriptor,
    uint256 maxSegmentCount
  )
    ERC721("Sablier V2 Lockup Dynamic NFT", "SAB-V2-LOCKUP-DYN")
    SablierV2Lockup(initialAdmin, initialComptroller, initialNFTDescriptor)
  {
    MAX_SEGMENT_COUNT = maxSegmentCount;
    nextStreamId = 1;
  }

```

#### Construction function parameters:

The constructor function accepts and initializes 4 parameters:

- address initialAdmin: This is the address of the initial Contract admin. This parameter is passed to the constructor of the `SablierV2Lockup.sol` Contract and get initialized.

- ISablierV2Comptroller: This is the interface of the `SablierV2Comptroller` contract. This interface defines methods to set the flashFee, protocolFee and toggleFlashAsset respectively.

  - `flashFee()`:This function retrieves the current global flash fee. As it's a view function, it doesn't modify any state but simply returns the current fee.

    - flashFee: This is a global fee that is applied to all flash loans made in the Sablier protocol. `Flash loans` are a feature in DeFi where users can borrow assets without collateral with the condition that the loan is returned within the same transaction.

  - `setFlashFee()`: This function allows the admin to update the flash fee charged on all flash loans made with any ERC-20 asset. It emits a `SetFlashFee` event which includes the old and new flash fee.

  - `setProtocolFee`: This function sets a new protocol fee that will be charged on all streams created with the provided ERC-20 asset. It emits a `SetProtocolFee` event.

    - protocolFees: This is a mapping that stores the protocol fee for each ERC-20 asset used in the streams of the Sablier protocol.
    - A stream in Sablier is a money streaming service where funds are transferred from a sender to a recipient over time.

  - `toggleFlashAsset()`: This function toggles the flash loanability of an ERC-20 asset. It emits a `ToggleFlashAsset` event.

```sh

import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import { UD60x18 } from "@prb/math/src/UD60x18.sol";
import { IAdminable } from "./IAdminable.sol";

interface ISablierV2Comptroller is IAdminable {

    event SetFlashFee(address indexed admin, UD60x18 oldFlashFee, UD60x18 newFlashFee);

    event SetProtocolFee(address indexed admin, IERC20 indexed asset, UD60x18 oldProtocolFee, UD60x18 newProtocolFee);

    event ToggleFlashAsset(address indexed admin, IERC20 indexed asset, bool newFlag);

    function flashFee() external view returns (UD60x18 fee);

    function isFlashAsset(IERC20 token) external view returns (bool result);

    function protocolFees(IERC20 asset) external view returns (UD60x18 fee);

    function setFlashFee(UD60x18 newFlashFee) external;

    function setProtocolFee(IERC20 asset, UD60x18 newProtocolFee) external;

    function toggleFlashAsset(IERC20 asset) external;
}

```

- ISablierV2NFTDescriptor: This is the interface of the `SablierV2NFTDescriptor` contract. This interface defines a method, that generates the Uniform Resource Identifier (URI) describing a particular stream NFT.

  - `tokenURI(IERC721Metadata sablier, uint256 streamId)`: This function generates a data URI that describes a specific stream NFT.
    The URI is compliant with the ERC721 metadata standard.
    The function takes in two parameters:
    - sablier: This is the address of the Sablier contract where the stream was created.
    - streamId: This is the id of the stream for which to produce a description. This Id is queried to return the URI that contains all the details about the NFT asset.

```sh


import { IERC721Metadata } from "@openzeppelin/contracts/token/ERC721/extensions/IERC721Metadata.sol";

interface ISablierV2NFTDescriptor {

    function tokenURI(IERC721Metadata sablier, uint256 streamId) external view returns (string memory uri);
}

```

- uint256 maxSegmentCount: This is maximum number of segments allowed in a stream.
  This is an immutable state variable hence it is initialized at construction time and can't be changed.
  By default the maximum number of Segment in a stream is 300 Segments.

```sh
uint256 public immutable override MAX_SEGMENT_COUNT;
```

### Public View Getter Functions

As of the time of writing this review (October 2023), the `SablierV2LockupDynamic` contract has 18 getter functions which uses the `streamId` to query the Blockchain state and returns the specific data stored in the Blockchain Storage.
The storage variables are the members defined in the `Stream` struct.

- getAsset(uint256 streamId)
- getDepositedAmount(uint256 streamId)
- getEndTime(uint256 streamId)
- getRange(uint256 streamId)
- getRefundedAmount(uint256 streamId)
- getSegments(uint256 streamId)
- getSender(uint256 streamId)
- getStartTime(uint256 streamId)
- getStream(uint256 streamId)
- getWithdrawnAmount(uint256 streamId)
- isCancelable(uint256 streamId)
- isTransferable(uint256 streamId)
- isDepleted(uint256 streamId)
- isStream(uint256 streamId)
- refundableAmountOf(uint256 streamId)
- statusOf(uint256 streamId)
- function streamedAmountOf(uint256 streamId)
- wasCanceled(uint256 streamId)

- getAsset(uint256 streamId): This is the function that queries the streamId and retrieves the asset from the Blockchain state. The assets is an ERC20-Token, used for the Streaming and defined in the Stream Struct. This function inherits and overrides the the modifier from the SablierV2Lockup abstract contract.
- - `notNull(streamId)`: This is modifier defined in the `SablierV2Lockup` abstract contract, `src/abstracts/SablierV2Lockup.sol`.
    It calls the `isStream` method defined in the interface of SablierV2Lockup contract, `src/interfaces/ISablierV2Lockup.sol` which checks and ensure that the streamId passed in as a parameter does not reference a null stream and if Null, it reverts using the custom error `SablierV2Lockup_Null(streamId)` defined in the `src/libraries/Errors.sol`

  ```sh
  modifier notNull(uint256 streamId) {
        if (!isStream(streamId)) {
            revert Errors.SablierV2Lockup_Null(streamId);
        }
        _;
    }

  ```

  ```sh
  function getAsset(uint256 streamId) external view override notNull(streamId) returns (IERC20 asset) {
    asset = _streams[streamId].asset;
  }
  ```

- getDepositedAmount(uint256 streamId): This is the function that queries the streamId and retrieves the depositedAmount from the Blockchain state.
  - Deposited amount is defined in the `Amounts` Struct in the `Lockup` library and referenced in the Stream struct in the Lockup library, `src/types/DataTypes.sol`.
  - Amounts Struct contains the deposit, withdrawn, and refunded amounts, all denoted in units of the asset's decimals.
  -

```sh
function getDepositedAmount(
    uint256 streamId
  ) external view override notNull(streamId) returns (uint128 depositedAmount) {
    depositedAmount = _streams[streamId].amounts.deposited;
  }
```

- getEndTime(uint256 streamId): This is the function that queries the streamId and retrieves the Unix timestamp of the stream's end time from the Blockchain state.
  - the endTime is defined in the Streams struct and referenced in the \_streams mapping
  - The `Unix` timestamp is the Epoch time. It measures the number of seconds that have elapsed since 00:00:00 UTC on 1 January 1970

```sh
 function getEndTime(uint256 streamId) external view override notNull(streamId) returns (uint40 endTime) {

    endTime = _streams[streamId].endTime;
  }

```

- getRange(uint256 streamId): This is the function that queries the streamId and retrieves the Unix timestamp of the stream's range from the Range struct.
  - The `Range `struct is defined in the `LockupDynamic` library, `src/types/DataTypes.sol`. the Range struct contains the time range for the stream's start and stream's end all of uint40 data type.

```sh
 struct Range {
    uint40 start;
    uint40 end;
  }
```

```sh
function getRange(
    uint256 streamId
  ) external view override notNull(streamId) returns (LockupDynamic.Range memory range) {

    range = LockupDynamic.Range({ start: _streams[streamId].startTime, end: _streams[streamId].endTime });
  }

```

- getRefundedAmount(uint256 streamId): This is the function that queries the streamId and retrieves refundedAmount from the Blockchain storage.
  - `amounts` is Struct containing the deposit, withdrawn, and refunded amounts, all denoted in units of the asset's decimals.
  - The 'amounts' struct is defined in the`Lockup` library and referenced in the `Stream` struct defined in the `LockupDynamic` library, `src/types/DataTypes.sol`

```sh
struct Amounts {
    // slot 0
    uint128 deposited;
    uint128 withdrawn;
    // slot 1
    uint128 refunded;
  }
```

```sh
function getRefundedAmount(
    uint256 streamId
  ) external view override notNull(streamId) returns (uint128 refundedAmount) {

    refundedAmount = _streams[streamId].amounts.refunded;
  }
```

- getSegments(uint256 streamId):This is the function that queries the streamId and retrieves segments.
  - Segment: The Segment helps to facilitate the calculation of the custom streaming curve, which determines how the payment rate varies over time.
  - The `Segment` is a struct defined in the LockupDynamic library and referenced in the `Stream` struct

```sh
struct Segment {
    // slot 0
    uint128 amount;
    UD2x18 exponent;
    uint40 milestone;
  }
```

```sh
function getSegments(
    uint256 streamId
  ) external view override notNull(streamId) returns (LockupDynamic.Segment[] memory segments) {
    segments = _streams[streamId].segments;
  }
```

- getSender(uint256 streamId): This is the function that queries the streamId and retrieves sender.
  - The `sender` is the Externally Owned Account(EOA)/ address, that created the assets (ERC20-Token) streaming.
  - The sender has the ability to cancel the stream.

```sh
 function getSender(uint256 streamId) external view override notNull(streamId) returns (address sender) {

    sender = _streams[streamId].sender;
  }

```

- getStartTime(uint256 streamId): This is the function that queries the streamId and retrieves the Unix timestamp of the stream startTime, which is of unit40 data type.

```sh
function getStartTime(uint256 streamId) external view override notNull(streamId) returns (uint40 startTime) {
    startTime = _streams[streamId].startTime;
  }

```

- getStream(uint256 streamId): This is the function that queries the streamId and retrieves stream from the Blockchain Storage. It gets all the data stored in the `Stream` struct.
  - Checks:
    - it calls the internal `_statusOf()` defined in the `SablierV2Lockup` abstract contract and checks if the status equal to the `SETTLED`.
    - SETTLED is a member of the enum `Status` defined in the `Lockup` library in the `src/types/DataTypes.sol` contract.
    - if the Stream state is SETTLED, then it sets the `isCancelable` to false, thus you a user cannot cancel the stream again. A SETTLED Stream cannot be canceled.

```sh
function _statusOf(uint256 streamId) internal view virtual returns (Lockup.Status);

```

- The enum `Status` has the following members:
  - PENDING shows that a Stream has been created but has not started, the assets (ERC20-Tokens) are in a pending state.
  - STREAMING shows an active stream where assets are currently being streamed.
  - SETTLED shows that all assets have been streamed and recipient is due to withdraw them.
  - CANCELED shows that a Stream has been Canceled and remaining assets awaits recipient's withdrawal.
  - DEPLETED shows that a stream has be used up and all assets have been withdrawn and/or refunded.

```sh
 enum Status {
    PENDING, // value 0
    STREAMING, // value 1
    SETTLED, // value 2
    CANCELED, // value 3
    DEPLETED // value 4
  }
```

```sh
function getStream(
    uint256 streamId
  ) external view override notNull(streamId) returns (LockupDynamic.Stream memory stream) {

    stream = _streams[streamId];

    if (_statusOf(streamId) == Lockup.Status.SETTLED) {
      stream.isCancelable = false;
    }
  }
```

- getWithdrawnAmount(uint256 streamId): This is the function that queries the streamId and retrieves withdrawn Amount.
  - `withdrawn` is defined as a member in the `Amounts` Struct in the `Lockup` library and referenced in the `Stream` struct in the `LockupDynamic` library, `src/types/DataTypes.sol`

```sh
function getWithdrawnAmount(
    uint256 streamId
  ) external view override notNull(streamId) returns (uint128 withdrawnAmount) {

    withdrawnAmount = _streams[streamId].amounts.withdrawn;
  }
```

- isCancelable(uint256 streamId): This is the function that queries the streamId and returns boolean value to know if the Stream can be canceled by a user.
  - Checks:
    - It checks to make sure that the current status of Stream is not `SETTLED`. This ensures that you cannot canceled a Stream that has been Settled.
    - Thus if the stream is `STREAMING`(active), then a user can cancel it by calling this function.

```sh

 function isCancelable(uint256 streamId) external view override notNull(streamId) returns (bool result) {

    if (_statusOf(streamId) != Lockup.Status.SETTLED) {
      result = _streams[streamId].isCancelable;
    }
  }
```

- isTransferable(uint256 streamId): This is the function that queries the streamId and returns a boolean value/ result that indicates if the stream NFT is transferable.
  - Whenever a new Stream is created and funded by `msg.sender` The Sablier Protocol wraps every stream in an ERC-721 non-fungible token (NFT), making the stream recipient the owner of the NFT. The recipient can transfer the NFT to another address, and this also transfers the right to withdraw funds from the stream, including any funds already streamed.

```sh
 function isTransferable(
    uint256 streamId
  ) public view override(ISablierV2Lockup, SablierV2Lockup) notNull(streamId) returns (bool result) {

    result = _streams[streamId].isTransferable;
  }


```

- isDepleted(uint256 streamId): This is the function that queries the streamId and returns a boolean value that indicates is the Stream is depleted/used up.

```sh
function isDepleted(
    uint256 streamId
  ) public view override(ISablierV2Lockup, SablierV2Lockup) notNull(streamId) returns (bool result) {

    result = _streams[streamId].isDepleted;
  }
```

- isStream(uint256 streamId): This is the function that queries the streamId and returns a boolean value that indicates if the struct entity/member exists

```sh
 function isStream(uint256 streamId) public view override(ISablierV2Lockup, SablierV2Lockup) returns (bool result) {

    result = _streams[streamId].isStream;
  }

```

- refundableAmountOf(uint256 streamId): This is the function that queries the streamId and retrieves refundableAmount.
  - Checks:
    - It checks if the `Stream` can be canceled and also checks to ensure that the Stream is not depleted by calling `isCancelable` and `isDepleted` member of the Stream Struct respectively.
    - If both conditions valid, then it calculates and returns the refundableAmount.
    - `_calculateStreamedAmount` is an internal function, it contains the logic for calculating the streamed amount.
    - If both conditions ( `isCancelable` and `isDepleted` ) are not valid, then the refundableAmount is Zero.

```sh
function refundableAmountOf(
    uint256 streamId
  ) external view override notNull(streamId) returns (uint128 refundableAmount) {

    if (_streams[streamId].isCancelable && !_streams[streamId].isDepleted) {
      refundableAmount = _streams[streamId].amounts.deposited - _calculateStreamedAmount(streamId);
    }

  }

```

- statusOf(uint256 streamId): This is the function that queries the streamId and returns the Status of Stream by calling the internal function `_statusOf(streamId)`
  - `_statusOf(streamId)` is the internal function that contains the logic that checks the Status of the Stream. The `Status` of the enum type with members as defined above.

```sh
function statusOf(uint256 streamId) external view override notNull(streamId) returns (Lockup.Status status) {

    status = _statusOf(streamId);
  }
```

```sh
function _statusOf(uint256 streamId) internal view override returns (Lockup.Status) {
    if (_streams[streamId].isDepleted) {
      return Lockup.Status.DEPLETED;
    } else if (_streams[streamId].wasCanceled) {
      return Lockup.Status.CANCELED;
    }

    if (block.timestamp < _streams[streamId].startTime) {
      return Lockup.Status.PENDING;
    }

    if (_calculateStreamedAmount(streamId) < _streams[streamId].amounts.deposited) {
      return Lockup.Status.STREAMING;
    } else {
      return Lockup.Status.SETTLED;
    }
  }
```

- function streamedAmountOf(uint256 streamId): This is the function that queries the streamId, and Calculates the amount streamed to the recipient by calling the internal `_streamedAmountOf(streamId)` and retrieves the streamed Amount.
  - When a Stream is cancelled, the amount streamed is calculated as the difference between the deposited amount and the refunded amount.
  - When a Stream is depleted/used up, the streamed amount is equivalent to the total amount withdrawn.
  - `_streamedAmountOf(streamId)` is the internal function that contains the logic for calculating the Streamed Amount.

```sh
  function streamedAmountOf(
    uint256 streamId
  ) public view override(ISablierV2Lockup, ISablierV2LockupDynamic) notNull(streamId) returns (uint128 streamedAmount) {

    streamedAmount = _streamedAmountOf(streamId);
  }
```

```sh
function _streamedAmountOf(uint256 streamId) internal view returns (uint128) {
    Lockup.Amounts memory amounts = _streams[streamId].amounts;

    if (_streams[streamId].isDepleted) {
      return amounts.withdrawn;
    } else if (_streams[streamId].wasCanceled) {
      return amounts.deposited - amounts.refunded;
    }

    return _calculateStreamedAmount(streamId);
  }
```

- wasCanceled(uint256 streamId): This is the function that queries the streamId and returns a boolean indicating if the stream was canceled.

```sh
 function wasCanceled(
    uint256 streamId
  ) public view override(ISablierV2Lockup, SablierV2Lockup) notNull(streamId) returns (bool result) {


    result = _streams[streamId].wasCanceled;
  }
```

### External functions

- createWithDeltas(LockupDynamic.CreateWithDeltas calldata params)
- createWithMilestones(LockupDynamic.CreateWithMilestones calldata params)

- createWithDeltas(LockupDynamic.CreateWithDeltas calldata params): This is the function that `SablierV2LockupDynamic` used to Create Stream (Money Stream) using Deltas, `src/SablierV2LockupDynamic.sol`.

  - It Creates a stream by setting the start time to `block.timestamp`, and the end time to the sum of
    `block.timestamp` and all specified time deltas.
  - It returns the `streamId` of the newly created Stream.
  - Whenever a new Stream is created and funded by the `msg.sender`, the Sablier Protocol wraps every stream in an `ERC-721 non-fungible token (NFT)`, making the stream recipient the owner of the NFT.
  - The recipient can transfer the NFT to another address, and this also transfers the right to withdraw funds from the stream, including any funds already streamed.
  - `delta` refers to a time difference in seconds between segments. delta is a part of the `SegmentWithDelta` struct used at runtime.
  - The `SegmentWithDelta` struct is used to create custom streaming curves for a Lockup Dynamic stream. The SegmentWithDelta struct is defined in the `LockupDynamic` library and referenced in the `CreateWithDeltas` struct, `src/types/DataTypes.sol`
  - `noDelegateCall` is a modifier defined in the `NoDelegateCall.sol` abstract contract. It prevents delegateCall, `src/abstracts/NoDelegateCall.sol`, hence delegate calls cannot be called on this function.
  - When this function is invoked, it calls the internal function, `_createWithMilestones` to create the Stream and returns the `streamId`, which can be used to query the Blockchain State and return any needed data about the Stream.

```sh
struct SegmentWithDelta {
    uint128 amount;
    UD2x18 exponent;
    uint40 delta;
  }
```

```sh
struct CreateWithDeltas {
    address sender;
    bool cancelable;
    bool transferable;
    address recipient;
    uint128 totalAmount;
    IERC20 asset;
    Broker broker;
    SegmentWithDelta[] segments;
  }

```

```sh
struct Broker {
  address account;
  UD60x18 fee;
}
```

```sh

  function createWithDeltas(
    LockupDynamic.CreateWithDeltas calldata params
  ) external override noDelegateCall returns (uint256 streamId) {

    LockupDynamic.Segment[] memory segments = Helpers.checkDeltasAndCalculateMilestones(params.segments);

    streamId = _createWithMilestones(
      LockupDynamic.CreateWithMilestones({
        asset: params.asset,
        broker: params.broker,
        cancelable: params.cancelable,
        transferable: params.transferable,
        recipient: params.recipient,
        segments: segments,
        sender: params.sender,
        startTime: uint40(block.timestamp),
        totalAmount: params.totalAmount
      })
    );
  }
```

- Where:

  - asset: Is the contract address of the ERC-20 asset used for streaming.
  - broker: Is Struct containing the address of the broker assisting in creating the stream and the percentage fee paid to the broker from the totalAmount
  - cancelable: Indicates if the stream can be canceled.
  - transferable: Indicates if the stream NFT is transferable.
  - recipient: Is the address receiving the streamed assets.
  - segments: Is used to compose the custom streaming curve
  - sender: The address streaming the assets and has the ability to cancel the stream. The sender doesn't have to be the same as `msg.sender`
  - totalAmount:The total amount of ERC-20 assets to be paid, including the stream deposit and any potential fees, all denoted in units of the asset's decimals.

- createWithMilestones(LockupDynamic.CreateWithMilestones calldata params): This is the function that `SablierV2LockupDynamic` used to Create Stream (Money Stream) using Milestones, `src/SablierV2LockupDynamic.sol`
  - When this function is invoked, it calls the the internal function, `_createWithMilestones(params)`, which contains the logic that allows user to create Stream (Money Stream) with milestones and returns the `streamId`, which can be used to query the Blockchain State and return any needed data about the Stream.
  - milestone is the time component of a segment, which itself is a component of a Lockup Dynamic stream. Milestones play a crucial role in the calculation of the custom streaming curve.
  -

```sh
function createWithMilestones(
    LockupDynamic.CreateWithMilestones calldata params
  ) external override noDelegateCall returns (uint256 streamId) {
    streamId = _createWithMilestones(params);
  }

```

### Internal View functions

- \_calculateStreamedAmount(uint256 streamId)

```sh
function _calculateStreamedAmount(uint256 streamId) internal view returns (uint128) {

    if (_streams[streamId].startTime >= currentTime) {
      return 0;
    }

    uint40 endTime = _streams[streamId].endTime;
    if (endTime <= currentTime) {
      return _streams[streamId].amounts.deposited;
    }

    if (_streams[streamId].segments.length > 1) {

      return _calculateStreamedAmountForMultipleSegments(streamId);
    } else {

      return _calculateStreamedAmountForOneSegment(streamId);
    }
  }
```

- \_calculateStreamedAmountForMultipleSegments(uint256 streamId)

```sh

function _calculateStreamedAmountForMultipleSegments(uint256 streamId) internal view returns (uint128) {
    unchecked {
      uint40 currentTime = uint40(block.timestamp);
      LockupDynamic.Stream memory stream = _streams[streamId];

      uint128 previousSegmentAmounts;
      uint40 currentSegmentMilestone = stream.segments[0].milestone;
      uint256 index = 0;
      while (currentSegmentMilestone < currentTime) {
        previousSegmentAmounts += stream.segments[index].amount;
        index += 1;
        currentSegmentMilestone = stream.segments[index].milestone;
      }

      SD59x18 currentSegmentAmount = stream.segments[index].amount.intoSD59x18();
      SD59x18 currentSegmentExponent = stream.segments[index].exponent.intoSD59x18();
      currentSegmentMilestone = stream.segments[index].milestone;

      uint40 previousMilestone;
      if (index > 0) {

        previousMilestone = stream.segments[index - 1].milestone;
      } else {

        previousMilestone = stream.startTime;
      }


      SD59x18 elapsedSegmentTime = (currentTime - previousMilestone).intoSD59x18();
      SD59x18 totalSegmentTime = (currentSegmentMilestone - previousMilestone).intoSD59x18();


      SD59x18 elapsedSegmentTimePercentage = elapsedSegmentTime.div(totalSegmentTime);

      SD59x18 multiplier = elapsedSegmentTimePercentage.pow(currentSegmentExponent);
      SD59x18 segmentStreamedAmount = multiplier.mul(currentSegmentAmount);


      if (segmentStreamedAmount.gt(currentSegmentAmount)) {
        return previousSegmentAmounts > stream.amounts.withdrawn ? previousSegmentAmounts : stream.amounts.withdrawn;
      }

      return previousSegmentAmounts + uint128(segmentStreamedAmount.intoUint256());
    }
  }


```

- \_calculateStreamedAmountForOneSegment(uint256 streamId)

```sh

function _calculateStreamedAmountForOneSegment(uint256 streamId) internal view returns (uint128) {
    unchecked {

      SD59x18 elapsedTime = (uint40(block.timestamp) - _streams[streamId].startTime).intoSD59x18();
      SD59x18 totalTime = (_streams[streamId].endTime - _streams[streamId].startTime).intoSD59x18();

      SD59x18 elapsedTimePercentage = elapsedTime.div(totalTime);

      SD59x18 exponent = _streams[streamId].segments[0].exponent.intoSD59x18();
      SD59x18 depositedAmount = _streams[streamId].amounts.deposited.intoSD59x18();

      SD59x18 multiplier = elapsedTimePercentage.pow(exponent);
      SD59x18 streamedAmount = multiplier.mul(depositedAmount);

      if (streamedAmount.gt(depositedAmount)) {
        return _streams[streamId].amounts.withdrawn;
      }

      return uint128(streamedAmount.intoUint256());
    }
  }

```

- \_isCallerStreamSender(uint256 streamId)

```sh
function _isCallerStreamSender(uint256 streamId) internal view override returns (bool) {
    return msg.sender == _streams[streamId].sender;
  }


  function _statusOf(uint256 streamId) internal view override returns (Lockup.Status) {
    if (_streams[streamId].isDepleted) {
      return Lockup.Status.DEPLETED;
    } else if (_streams[streamId].wasCanceled) {
      return Lockup.Status.CANCELED;
    }

    if (block.timestamp < _streams[streamId].startTime) {
      return Lockup.Status.PENDING;
    }

    if (_calculateStreamedAmount(streamId) < _streams[streamId].amounts.deposited) {
      return Lockup.Status.STREAMING;
    } else {
      return Lockup.Status.SETTLED;
    }
  }

```

- \_streamedAmountOf(uint256 streamId)

```sh

function _streamedAmountOf(uint256 streamId) internal view returns (uint128) {
    Lockup.Amounts memory amounts = _streams[streamId].amounts;

    if (_streams[streamId].isDepleted) {
      return amounts.withdrawn;
    } else if (_streams[streamId].wasCanceled) {
      return amounts.deposited - amounts.refunded;
    }

    return _calculateStreamedAmount(streamId);
  }





```

- \_withdrawableAmountOf(uint256 streamId)

```sh
function _withdrawableAmountOf(uint256 streamId) internal view override returns (uint128) {
    return _streamedAmountOf(streamId) - _streams[streamId].amounts.withdrawn;
  }
```

- \_cancel(uint256 streamId)

```sh


function _cancel(uint256 streamId) internal override {
    // Calculate the streamed amount.
    uint128 streamedAmount = _calculateStreamedAmount(streamId);

    // Retrieve the amounts from storage.
    Lockup.Amounts memory amounts = _streams[streamId].amounts;

    // Checks: the stream is not settled.
    if (streamedAmount >= amounts.deposited) {
      revert Errors.SablierV2Lockup_StreamSettled(streamId);
    }

    // Checks: the stream is cancelable.
    if (!_streams[streamId].isCancelable) {
      revert Errors.SablierV2Lockup_StreamNotCancelable(streamId);
    }

    // Calculate the sender's and the recipient's amount.
    uint128 senderAmount = amounts.deposited - streamedAmount;
    uint128 recipientAmount = streamedAmount - amounts.withdrawn;

    // Effects: mark the stream as canceled.
    _streams[streamId].wasCanceled = true;

    // Effects: make the stream not cancelable anymore, because a stream can only be canceled once.
    _streams[streamId].isCancelable = false;

    // Effects: If there are no assets left for the recipient to withdraw, mark the stream as depleted.
    if (recipientAmount == 0) {
      _streams[streamId].isDepleted = true;
    }

    // Effects: set the refunded amount.
    _streams[streamId].amounts.refunded = senderAmount;

    // Retrieve the sender and the recipient from storage.
    address sender = _streams[streamId].sender;
    address recipient = _ownerOf(streamId);

    // Interactions: refund the sender.
    _streams[streamId].asset.safeTransfer({ to: sender, value: senderAmount });


    if (msg.sender == sender) {
      if (recipient.code.length > 0) {
        try
          ISablierV2LockupRecipient(recipient).onStreamCanceled({
            streamId: streamId,
            sender: sender,
            senderAmount: senderAmount,
            recipientAmount: recipientAmount
          })
        {} catch {}
      }
    }

    else {
      if (sender.code.length > 0) {
        try
          ISablierV2LockupSender(sender).onStreamCanceled({
            streamId: streamId,
            recipient: recipient,
            senderAmount: senderAmount,
            recipientAmount: recipientAmount
          })
        {} catch {}
      }
    }

    // Log the cancellation.
    emit ISablierV2Lockup.CancelLockupStream(streamId, sender, recipient, senderAmount, recipientAmount);
  }


```

- \_createWithMilestones

```sh


function _createWithMilestones(LockupDynamic.CreateWithMilestones memory params) internal returns (uint256 streamId) {

    UD60x18 protocolFee = comptroller.protocolFees(params.asset);

    // Checks: check the fees and calculate the fee amounts.
    Lockup.CreateAmounts memory createAmounts = Helpers.checkAndCalculateFees(
      params.totalAmount,
      protocolFee,
      params.broker.fee,
      MAX_FEE
    );

    // Checks: validate the user-provided parameters.
    Helpers.checkCreateWithMilestones(createAmounts.deposit, params.segments, MAX_SEGMENT_COUNT, params.startTime);

    // Load the stream id in a variable.
    streamId = nextStreamId;

    // Effects: create the stream.
    LockupDynamic.Stream storage stream = _streams[streamId];
    stream.amounts.deposited = createAmounts.deposit;
    stream.asset = params.asset;
    stream.isCancelable = params.cancelable;
    stream.isTransferable = params.transferable;
    stream.isStream = true;
    stream.sender = params.sender;

    unchecked {
      // The segment count cannot be zero at this point.
      uint256 segmentCount = params.segments.length;
      stream.endTime = params.segments[segmentCount - 1].milestone;
      stream.startTime = params.startTime;


      for (uint256 i = 0; i < segmentCount; ++i) {
        stream.segments.push(params.segments[i]);
      }

      // Effects: bump the next stream id and record the protocol fee.
      // Using unchecked arithmetic because these calculations cannot realistically overflow, ever.
      nextStreamId = streamId + 1;
      protocolRevenues[params.asset] = protocolRevenues[params.asset] + createAmounts.protocolFee;
    }

    // Effects: mint the NFT to the recipient.
    _mint({ to: params.recipient, tokenId: streamId });


    unchecked {
      params.asset.safeTransferFrom({
        from: msg.sender,
        to: address(this),
        value: createAmounts.deposit + createAmounts.protocolFee
      });
    }

    // Interactions: pay the broker fee, if not zero.
    if (createAmounts.brokerFee > 0) {
      params.asset.safeTransferFrom({ from: msg.sender, to: params.broker.account, value: createAmounts.brokerFee });
    }

    // Log the newly created stream.
    emit ISablierV2LockupDynamic.CreateLockupDynamicStream({
      streamId: streamId,
      funder: msg.sender,
      sender: params.sender,
      recipient: params.recipient,
      amounts: createAmounts,
      asset: params.asset,
      cancelable: params.cancelable,
      transferable: params.transferable,
      segments: params.segments,
      range: LockupDynamic.Range({ start: stream.startTime, end: stream.endTime }),
      broker: params.broker.account
    });
  }


```

- \_renounce(uint256 streamId)

```sh

function _renounce(uint256 streamId) internal override {
    // Checks: the stream is cancelable.
    if (!_streams[streamId].isCancelable) {
      revert Errors.SablierV2Lockup_StreamNotCancelable(streamId);
    }

    // Effects: renounce the stream by making it not cancelable.
    _streams[streamId].isCancelable = false;
  }



```

- \_withdraw(uint256 streamId, address to, uint128 amount)

```sh
function _withdraw(uint256 streamId, address to, uint128 amount) internal override {
    // Effects: update the withdrawn amount.
    _streams[streamId].amounts.withdrawn = _streams[streamId].amounts.withdrawn + amount;

    // Retrieve the amounts from storage.
    Lockup.Amounts memory amounts = _streams[streamId].amounts;

    if (amounts.withdrawn >= amounts.deposited - amounts.refunded) {
      // Effects: mark the stream as depleted.
      _streams[streamId].isDepleted = true;

      // Effects: make the stream not cancelable anymore, because a depleted stream cannot be canceled.
      _streams[streamId].isCancelable = false;
    }

    // Interactions: perform the ERC-20 transfer.
    _streams[streamId].asset.safeTransfer({ to: to, value: amount });
  }

```
