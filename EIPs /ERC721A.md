## [RC721A](https://github.com/chiru-labs/ERC721A/blob/main/contracts/ERC721A.sol)
### 状态变量
#### **`_packedOwnerships`**
```
// Mapping from token ID to ownership details
mapping(uint256 => uint256) private _packedOwnerships;
```
位区间 (Bits) | 字段名|长度|作用说明
|------------------|----------|--------------|-----------------|
|[0..159]|addr|160位|拥有者地址。以太坊地址正好是 160 位（20字节）。
|[160..223]|startTimestamp|64位|铸造/转移的时间戳。用于记录这个 NFT 是什么时候被谁拿到的（可用于计算持有时间）。
|[224]|burned|1位|是否已销毁。0 表示正常|1 表示已进入黑洞地址。
|[225]|nextInitialized|1位|关键标志位。用于告诉合约“下一个 Token ID 是否已经有明确的拥有者数据”。
|[226..231]|(Reserved)|6位|预留空间|暂时未使用。
|[232..255]|extraData|24位|自定义额外数据。开发者可以利用这 3 个字节存储一些小信息（如：NFT 的某种等级）。
#### **`_packedAddressData`**
```
// Mapping owner address to address data.
mapping(address => uint256) private _packedAddressData;
```
位区间 (Bits) | 字段名|长度|作用说明
|------------------|----------|--------------|-----------------|
|[0..63]|balance|64位|持有余额。用户当前手里握着多少个该系列的 NFT。
|[64..127]|numberMinted|64位|累计铸造量。用户历史上总共 Mint 了多少个（哪怕卖掉了|这个数也不会减）。用于做限购判断。
|[128..191]|numberBurned|64位|累计销毁量。用户总共销毁了多少个 NFT。
|[192..255]|aux|64位|辅助数据 (Auxiliary)。留给开发者的“自由发挥空间，你可以存任何与地址相关的数据。
### 函数
#### **`_mint`**
``` solidity
function _mint(address to, uint256 quantity) internal virtual {
        uint256 startTokenId = _currentIndex;
        if (quantity == 0) _revert(MintZeroQuantity.selector);
        // 1. 铸造前钩子            
        _beforeTokenTransfers(address(0), to, startTokenId, quantity);

        // Overflows are incredibly unrealistic.
        // `balance` and `numberMinted` have a maximum limit of 2**64.
        // `tokenId` has a maximum limit of 2**256.
        unchecked {
            // 2. 更新 NFT 拥有权数据:
            // - `address` to the owner.
            // - `startTimestamp` to the timestamp of minting.
            // - `burned` to `false`.
            // - `nextInitialized` to `quantity == 1`.  如果只铸造 1 个，标记下一条记录已经初始化。
            _packedOwnerships[startTokenId] = _packOwnershipData(
                to,
                _nextInitializedFlag(quantity) | _nextExtraData(address(0), to, 0)
            );

            // 3. 更新地址相关数据:
            // - `balance += quantity`.
            // - `numberMinted += quantity`.
            //
            // We can directly add to the `balance` and `numberMinted`.
            _packedAddressData[to] += quantity * ((1 << _BITPOS_NUMBER_MINTED) | 1);

            // 4. 检查接收者地址有效性.
            // Mask `to` to the lower 160 bits, in case the upper bits somehow aren't clean.
            uint256 toMasked = uint256(uint160(to)) & _BITMASK_ADDRESS;

            if (toMasked == 0) _revert(MintToZeroAddress.selector);

            uint256 end = startTokenId + quantity;
            uint256 tokenId = startTokenId;
            // 5. 检查连续铸造上限
            if (end - 1 > _sequentialUpTo()) _revert(SequentialMintExceedsLimit.selector);
            // 6. 发送事件
            do {
                assembly {
                    // Emit the `Transfer` event.
                    log4(
                        0, // Start of data (0, since no data).
                        0, // End of data (0, since no data).
                        _TRANSFER_EVENT_SIGNATURE, // Signature.
                        0, // `address(0)`.
                        toMasked, // `to`.
                        tokenId // `tokenId`.
                    )
                }
                // The `!=` check ensures that large values of `quantity`
                // that overflows uint256 will make the loop run out of gas.
            } while (++tokenId != end);
           // 7. 更新当前index
            _currentIndex = end;
        }
        // 8. 锻造后钩子
        _afterTokenTransfers(address(0), to, startTokenId, quantity);
    }

```

#### **`_packedOwnershipOf`**
``` solidity
// tokenId所有权溯源
function _packedOwnershipOf(uint256 tokenId) private view returns (uint256 packed) {
        // 1.边界检查
        if (_startTokenId() <= tokenId) {
            packed = _packedOwnerships[tokenId];
            // 2. 检查非连续 ID
            if (tokenId > _sequentialUpTo()) {
                if (_packedOwnershipExists(packed)) return packed;
                _revert(OwnerQueryForNonexistentToken.selector);
            }

            // 当前tokenId没有所有权信息，则向上溯源
            // If the data at the starting slot does not exist, start the scan.
            if (packed == 0) {
                if (tokenId >= _currentIndex) _revert(OwnerQueryForNonexistentToken.selector);
                // Invariant:
                // There will always be an initialized ownership slot
                // (i.e. `ownership.addr != address(0) && ownership.burned == false`)
                // before an unintialized ownership slot
                // (i.e. `ownership.addr == address(0) && ownership.burned == false`)
                // Hence, `tokenId` will not underflow.
                //
                // We can directly compare the packed value.
                // If the address is zero, packed will be zero.
                for (;;) {  // 开启无限循环，往回找
                    unchecked {
                        packed = _packedOwnerships[--tokenId];  // ID 一直减 1
                    }
                    if (packed == 0) continue;
                    // 找到了第一个不为 0 的位置
                    if (packed & _BITMASK_BURNED == 0) return packed;
                    // Otherwise, the token is burned, and we must revert.
                    // This handles the case of batch burned tokens, where only the burned bit
                    // of the starting slot is set, and remaining slots are left uninitialized.
                    _revert(OwnerQueryForNonexistentToken.selector);
                }
            }
            // Otherwise, the data exists and we can skip the scan.
            // This is possible because we have already achieved the target condition.
            // This saves 2143 gas on transfers of initialized tokens.
            // If the token is not burned, return `packed`. Otherwise, revert.
            if (packed & _BITMASK_BURNED == 0) return packed;
        }
        _revert(OwnerQueryForNonexistentToken.selector);
    }
```

#### **`transferFrom`**
``` solidity
function transferFrom(
        address from,
        address to,
        uint256 tokenId
    ) public payable virtual override {
        // 1. 调用_packedOwnershipOf, 找出tokenId所有权信息
        uint256 prevOwnershipPacked = _packedOwnershipOf(tokenId);

        // 2. Mask `from` to the lower 160 bits, in case the upper bits somehow aren't clean.
        from = address(uint160(uint256(uint160(from)) & _BITMASK_ADDRESS));

        if (address(uint160(prevOwnershipPacked)) != from) _revert(TransferFromIncorrectOwner.selector);

        (uint256 approvedAddressSlot, address approvedAddress) = _getApprovedSlotAndAddress(tokenId);

        // 3. 校验调用者权限
        // The nested ifs save around 20+ gas over a compound boolean condition.
        if (!_isSenderApprovedOrOwner(approvedAddress, from, _msgSenderERC721A()))
            if (!isApprovedForAll(from, _msgSenderERC721A())) _revert(TransferCallerNotOwnerNorApproved.selector);

        // 4. beforeTokenTransfers hook
        _beforeTokenTransfers(from, to, tokenId, 1);

        // 5. Clear approvals from the previous owner.
        assembly {
            if approvedAddress {
                // This is equivalent to `delete _tokenApprovals[tokenId]`.
                sstore(approvedAddressSlot, 0)
            }
        }

        // 6. balance & ownership 更新
        // Underflow of the sender's balance is impossible because we check for
        // ownership above and the recipient's balance can't realistically overflow.
        // Counter overflow is incredibly unrealistic as `tokenId` would have to be 2**256.
        unchecked {
            // We can directly increment and decrement the balances.
            --_packedAddressData[from]; // Updates: `balance -= 1`.
            ++_packedAddressData[to]; // Updates: `balance += 1`.

            // Updates:
            // - `address` to the next owner.
            // - `startTimestamp` to the timestamp of transfering.
            // - `burned` to `false`.
            // - `nextInitialized` to `true`.
            _packedOwnerships[tokenId] = _packOwnershipData(
                to,
                _BITMASK_NEXT_INITIALIZED | _nextExtraData(from, to, prevOwnershipPacked)
            );

            // If the next slot may not have been initialized (i.e. `nextInitialized == false`) .
            if (prevOwnershipPacked & _BITMASK_NEXT_INITIALIZED == 0) {
                uint256 nextTokenId = tokenId + 1;
                // If the next slot's address is zero and not burned (i.e. packed value is zero).
                if (_packedOwnerships[nextTokenId] == 0) {
                    // If the next slot is within bounds.
                    if (nextTokenId != _currentIndex) {
                        // Initialize the next slot to maintain correctness for `ownerOf(tokenId + 1)`.
                        _packedOwnerships[nextTokenId] = prevOwnershipPacked;
                    }
                }
            }
        }
        
        // Mask `to` to the lower 160 bits, in case the upper bits somehow aren't clean.
        uint256 toMasked = uint256(uint160(to)) & _BITMASK_ADDRESS;
        assembly {
            // 7. Emit the `Transfer` event.
            log4(
                0, // Start of data (0, since no data).
                0, // End of data (0, since no data).
                _TRANSFER_EVENT_SIGNATURE, // Signature.
                from, // `from`.
                toMasked, // `to`.
                tokenId // `tokenId`.
            )
        }
        if (toMasked == 0) _revert(TransferToZeroAddress.selector);

        // afterTokenTransfers hook
        _afterTokenTransfers(from, to, tokenId, 1);
    }
```
