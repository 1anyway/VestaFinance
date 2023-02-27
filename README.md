# VestaFinance
This is a repository where i store my research on VestaFinance project

## Vesta Finance Architecture
![image](https://github.com/1anyway/VestaFinance/blob/main/img/VestaFinance.png)

## The Steps to Opening a Trove
When borrowing&minting happen, a trove opened. When a borrower depositing collaterals and minting VST coins, that opens a trove.
1. initialize the parameters ```vestaParams.sanitizeParameters(_asset)```
2. Get the value of some needed parameters such as the price, fee, ICR and so on.
3. Add status, settings to the trove.
4. Insert the trove to sortedTroves and add trove owner to array.
   
The code of these operations is below
```solidity
function openTrove(
		address _asset,
		uint256 _tokenAmount,
		uint256 _maxFeePercentage,
		uint256 _VSTAmount,
		address _upperHint,
		address _lowerHint
	) external payable override {
        // The object vestaParams is appearing in VestaBase.sol file
		// Initialize parameters first  to opening a trove, it's something like you borrowing money from a bank, but also it's different. 
		vestaParams.sanitizeParameters(_asset);

		ContractsCache memory contractsCache = ContractsCache(
			troveManager,
			vestaParams.activePool(),
			VSTToken
		);
		// It's a structure, including the parameters setting and operations to open a trove.
		LocalVariables_openTrove memory vars;
		vars.asset = _asset;

		// Get amount that the asset you've deposited, asset can be ETH or others.
		_tokenAmount = getMethodValue(vars.asset, _tokenAmount, false);
		vars.price = vestaParams.priceFeed().fetchPrice(vars.asset);
		// Max fee percentage must be between 0.5% and 100%.
		_requireValidMaxFeePercentage(vars.asset, _maxFeePercentage);
		// That means you can't opening two of the same troves.
		_requireTroveisNotActive(vars.asset, contractsCache.troveManager, msg.sender);

		vars.VSTFee;
		vars.netDebt = _VSTAmount;

		// Mint VST
		vars.VSTFee = _triggerBorrowingFee(
			vars.asset,
			contractsCache.troveManager,
			contractsCache.VSTToken,
			_VSTAmount,
			_maxFeePercentage
		);

		// Add to the borrower the same amount of debt as the minted VST
		vars.netDebt = vars.netDebt.add(vars.VSTFee);
		_requireAtLeastMinNetDebt(vars.asset, vars.netDebt);

		// ICR is based on the composite debt, i.e. the requested VST amount + VST borrowing fee + VST gas comp.
		vars.compositeDebt = _getCompositeDebt(vars.asset, vars.netDebt);

		assert(vars.compositeDebt > 0);
		_requireLowerOrEqualsToVSTLimit(vars.asset, vars.compositeDebt);

		vars.ICR = VestaMath._computeCR(_tokenAmount, vars.compositeDebt, vars.price);
		vars.NICR = VestaMath._computeNominalCR(_tokenAmount, vars.compositeDebt);

		_requireICRisAboveMCR(vars.asset, vars.ICR);

		// Set the trove struct's properties
		contractsCache.troveManager.setTroveStatus(vars.asset, msg.sender, 1);
		contractsCache.troveManager.increaseTroveColl(vars.asset, msg.sender, _tokenAmount);
		contractsCache.troveManager.increaseTroveDebt(vars.asset, msg.sender, vars.compositeDebt);

		contractsCache.troveManager.updateTroveRewardSnapshots(vars.asset, msg.sender);
		vars.stake = contractsCache.troveManager.updateStakeAndTotalStakes(vars.asset, msg.sender);

		sortedTroves.insert(vars.asset, msg.sender, vars.NICR, _upperHint, _lowerHint);
		vars.arrayIndex = contractsCache.troveManager.addTroveOwnerToArray(vars.asset, msg.sender);
		emit TroveCreated(vars.asset, msg.sender, vars.arrayIndex);

		// Move the ether to the Active Pool, and mint the VSTAmount to the borrower
		_activePoolAddColl(vars.asset, contractsCache.activePool, _tokenAmount);
		_withdrawVST(
			vars.asset,
			contractsCache.activePool,
			contractsCache.VSTToken,
			msg.sender,
			_VSTAmount,
			vars.netDebt
		);
		// Move the VST gas compensation to the Gas Pool
		_withdrawVST(
			vars.asset,
			contractsCache.activePool,
			contractsCache.VSTToken,
			gasPoolAddress,
			vestaParams.VST_GAS_COMPENSATION(vars.asset),
			vestaParams.VST_GAS_COMPENSATION(vars.asset)
		);

		emit TroveUpdated(
			vars.asset,
			msg.sender,
			vars.compositeDebt,
			_tokenAmount,
			vars.stake,
			BorrowerOperation.openTrove
		);
		emit VSTBorrowingFeePaid(vars.asset, msg.sender, vars.VSTFee);
	}
```
## The Steps to Adjusting and Closing a Trove
### Adjusting a Trove
1. Get price from oracle.
2. Calculate rewards that have accumulated.
3. Update trove's coll and debt based on whether they increase or decrease.
4. The trove is re-ordered and inserted.
5. Change tokens and ETH from adjustment.
```solidity
	function _adjustTrove(
		address _asset,
		uint256 _assetSent,
		address _borrower,
		uint256 _collWithdrawal,
		uint256 _VSTChange,
		bool _isDebtIncrease,
		address _upperHint,
		address _lowerHint,
		uint256 _maxFeePercentage
	) internal {
		ContractsCache memory contractsCache = ContractsCache(
			troveManager,
			vestaParams.activePool(),
			VSTToken
		);
		LocalVariables_adjustTrove memory vars;
		vars.asset = _asset;

		if (_isDebtIncrease) {
			_requireLowerOrEqualsToVSTLimit(vars.asset, _VSTChange);
		}

		require(
			msg.value == 0 || msg.value == _assetSent,
			"BorrowerOp: _AssetSent and Msg.value aren't the same!"
		);

		vars.price = vestaParams.priceFeed().fetchPrice(vars.asset);

		if (_isDebtIncrease) {
			_requireValidMaxFeePercentage(vars.asset, _maxFeePercentage);
			_requireNonZeroDebtChange(_VSTChange);
		}
		require(
			_collWithdrawal == 0 || _assetSent == 0,
			"BorrowerOperations: Cannot withdraw and add coll"
		);

		_requireNonZeroAdjustment(_collWithdrawal, _VSTChange, _assetSent);
		_requireTroveisActive(vars.asset, contractsCache.troveManager, _borrower);

		// Confirm the operation is either a borrower adjusting their own trove, or a pure ETH transfer from the Stability Pool to a trove
		assert(
			msg.sender == _borrower ||
				(stabilityPoolManager.isStabilityPool(msg.sender) && _assetSent > 0 && _VSTChange == 0)
		);

		contractsCache.troveManager.applyPendingRewards(vars.asset, _borrower);

		// Get the collChange based on whether or not ETH was sent in the transaction
		(vars.collChange, vars.isCollIncrease) = _getCollChange(_assetSent, _collWithdrawal);

		vars.netDebtChange = _VSTChange;

		// If the adjustment incorporates a debt increase and system is in Normal Mode, then trigger a borrowing fee.
		if (_isDebtIncrease) {
			vars.VSTFee = _triggerBorrowingFee(
				vars.asset,
				contractsCache.troveManager,
				contractsCache.VSTToken,
				_VSTChange,
				_maxFeePercentage
			);
			vars.netDebtChange = vars.netDebtChange.add(vars.VSTFee); // The raw debt change includes the fee
		}

		vars.debt = contractsCache.troveManager.getTroveDebt(vars.asset, _borrower);
		vars.coll = contractsCache.troveManager.getTroveColl(vars.asset, _borrower);

		// Get the trove's old ICR before the adjustment, and what its new ICR will be after the adjustment
		vars.oldICR = VestaMath._computeCR(vars.coll, vars.debt, vars.price);
		vars.newICR = _getNewICRFromTroveChange(
			vars.coll,
			vars.debt,
			vars.collChange,
			vars.isCollIncrease,
			vars.netDebtChange,
			_isDebtIncrease,
			vars.price
		);
		require(
			_collWithdrawal <= vars.coll,
			"BorrowerOp: Trying to remove more than the trove holds"
		);

		// Check the adjustment satisfies all conditions for the current system mode
		_requireICRisAboveMCR(_asset, vars.newICR);

		// When the adjustment is a debt repayment, check it's a valid amount and that the caller has enough VST
		if (!_isDebtIncrease && _VSTChange > 0) {
			_requireAtLeastMinNetDebt(
				vars.asset,
				_getNetDebt(vars.asset, vars.debt).sub(vars.netDebtChange)
			);

			require(
				vars.netDebtChange <= vars.debt.sub(vestaParams.VST_GAS_COMPENSATION(vars.asset)),
				"BorrowerOps: Amount repaid must not be larger than the Trove's debt"
			);

			_requireSufficientVSTBalance(contractsCache.VSTToken, _borrower, vars.netDebtChange);
		}

		(vars.newColl, vars.newDebt) = _updateTroveFromAdjustment(
			vars.asset,
			contractsCache.troveManager,
			_borrower,
			vars.collChange,
			vars.isCollIncrease,
			vars.netDebtChange,
			_isDebtIncrease
		);
		vars.stake = contractsCache.troveManager.updateStakeAndTotalStakes(vars.asset, _borrower);

		// Re-insert trove in to the sorted list
		uint256 newNICR = _getNewNominalICRFromTroveChange(
			vars.coll,
			vars.debt,
			vars.collChange,
			vars.isCollIncrease,
			vars.netDebtChange,
			_isDebtIncrease
		);
		sortedTroves.reInsert(vars.asset, _borrower, newNICR, _upperHint, _lowerHint);

		emit TroveUpdated(
			vars.asset,
			_borrower,
			vars.newDebt != 0 ? vars.newDebt : vars.debt,
			vars.newColl != 0 ? vars.newColl : vars.coll,
			vars.stake,
			BorrowerOperation.adjustTrove
		);
		emit VSTBorrowingFeePaid(vars.asset, msg.sender, vars.VSTFee);

		// Use the unmodified _VSTChange here, as we don't send the fee to the user
		_moveTokensAndETHfromAdjustment(
			vars.asset,
			contractsCache.activePool,
			contractsCache.VSTToken,
			msg.sender,
			vars.collChange,
			vars.isCollIncrease,
			_VSTChange,
			_isDebtIncrease,
			vars.netDebtChange
		);
	}
```
### Closing a Trove
Trove will closed when it's owner repaying all it's collaterals, also when all of the owner's collaterals are liquidated or redeemed.
1. Calculate rewards that have accumulated.
2. Get parameters of trove and check.
3. Change active Pool and trove.
4. Update interest.
5. Clear trove.
6. Remove trove owner and remove trove from sorted troves.
7. Burn the repaid VST from the user's balance and the gas compensation from the Gas Pool.
   
 
```solidity
function _closeTrove(
		address _asset,
		uint256 _amountIn,
		IVestaDexTrader.ManualExchange[] memory _manualExchange
	) internal {
		if (_asset == WETH) _asset = ETH_REF_ADDRESS;

		ITroveManager troveManagerCached = troveManager;
		IActivePool activePoolCached = vestaParams.activePool();
		IVSTToken VSTTokenCached = VSTToken;

		_requireTroveisActive(_asset, troveManagerCached, msg.sender);

		//  Calculate rewards that have accumulated
		troveManagerCached.applyPendingRewards(_asset, msg.sender);
		uint256 coll = troveManagerCached.getTroveColl(_asset, msg.sender);
		uint256 debt = troveManagerCached.getTroveDebt(_asset, msg.sender);

		uint256 userVST = VSTTokenCached.balanceOf(msg.sender);

		if (debt > userVST) {
			uint256 amountNeeded = debt - userVST;
			address tokenIn = _asset;

			require(
				_manualExchange.length > 0,
				"BorrowerOps: Caller doesnt have enough VST to make repayment"
			);

			if (_amountIn == 0) {
				_amountIn = dexTrader.getAmountIn(amountNeeded, _manualExchange);
			}
			
			activePoolCached.unstake(_asset, msg.sender, _amountIn);
			activePoolCached.sendAsset(_asset, address(this), _amountIn);
			troveManagerCached.decreaseTroveColl(_asset, msg.sender, _amountIn);
			coll -= _amountIn;

			if (_asset == ETH_REF_ADDRESS) {
				tokenIn = WETH;
				IWETH(WETH).deposit{ value: _amountIn }();
			}

			IERC20Upgradeable(tokenIn).safeApprove(address(dexTrader), _amountIn);

			dexTrader.exchange(msg.sender, tokenIn, _amountIn, _manualExchange);

			require(
				VSTTokenCached.balanceOf(msg.sender) >= debt,
				"AutoSwapping Failed, Try increasing slippage inside ManualExchange"
			);
		}

		_requireSufficientVSTBalance(
			VSTTokenCached,
			msg.sender,
			debt.sub(vestaParams.VST_GAS_COMPENSATION(_asset))
		);

		troveManagerCached.removeStake(_asset, msg.sender);
		troveManagerCached.closeTrove(_asset, msg.sender);

		emit TroveUpdated(_asset, msg.sender, 0, 0, 0, BorrowerOperation.closeTrove);

		// Burn the repaid VST from the user's balance and the gas compensation from the Gas Pool
		_repayVST(_asset, activePoolCached, VSTTokenCached, msg.sender, debt);

		// Send the collateral back to the user
		activePoolCached.sendAsset(_asset, msg.sender, coll);
	}
```
And this is the closeTrove function in TroveManager.sol file
```solidity
function _closeTrove(
		address _asset,
		address _borrower,
		Status closedStatus
	) internal {
		assert(closedStatus != Status.nonExistent && closedStatus != Status.active);

		uint256 TroveOwnersArrayLength = TroveOwners[_asset].length;

		interestManager.exit(_asset, _borrower);

		uint256 totalInterest = userUnpaidInterest[_borrower][_asset];

		emit VaultUnpaidInterestUpdated(_asset, _borrower, 0);
		emit SystemUnpaidInterestUpdated(_asset, unpaidInterest[_asset] -= totalInterest);

		delete userUnpaidInterest[_borrower][_asset];

		vestaParams.activePool().unstake(_asset, _borrower, Troves[_borrower][_asset].coll);

		Troves[_borrower][_asset].status = closedStatus;
		Troves[_borrower][_asset].coll = 0;
		Troves[_borrower][_asset].debt = 0;

		rewardSnapshots[_borrower][_asset].asset = 0;
		rewardSnapshots[_borrower][_asset].VSTDebt = 0;

		_removeTroveOwner(_asset, _borrower, TroveOwnersArrayLength);
		sortedTroves.remove(_asset, _borrower);
	}
```

## The Mechanisms Behind of It
