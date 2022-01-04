
BitBay Dynamic Peg - Integration Guide for Exchanges
=====

1. [Chapter 1: The Dynamic Peg](#chapter-1-dynamic-peg)
2. [Chapter 2: Data handling on Exchanges](#chapter-2-data-handling-on-exchange)
3. [Chapter 3: Keeping peginfo in sync](#chapter-3-keeping-peginfo-in-sync)
4. [Chapter 4: Deposits](#chapter-4-deposits)
5. [Chapter 5: Keep balances in sync with peg](#chapter-5-keeping-balances-in-sync-with-peg)
6. [Chapter 6: Trades](#chapter-6-trades)
7. [Chapter 7: Sell Order management](#chapter-7-sell-orders-management)
8. [Chapter 8: Withdrawals](#chapter-8-withdrawals)
9. [Appendix 1: Overview of calls](#appendix-1-overview-of-calls)
10. [Appendix 2: Building bitbayd](#appendix-2-building-bitbayd)

Chapter 1: The Dynamic Peg
-----

The **Dynamic Peg** is a 'stablecoin' protocol that works on top of the UTXO system. It adds additional requirements for transactions. All coins operable in the system belong either to **RESERVE** or **LIQUIDITY**. There are protocol rules how liquid and reserve coins can be sent.

The minimum value of a single coin in Bitcoin is known as **1_Satoshi**.
In the BAY Dynamic Peg system, all coins are organized into 1200 structured columns, each column having its own ‘index’ number from 0-1199.
We call each indexed coin **1_Baytoshi**.

![Columns](coins1.png?raw=true "Columns")

Newly minted coins have an exponential distribution:

![NewCoins](coins3.png?raw=true "NewCoins")

In the BITBAY Qt Wallet you can browse newly minted coins with their details:

![NewCoins](coins2.png?raw=true "NewCoins")

*Above is an example of 10 minted BAY coins. The first column is 1% of 10 BAY, the second column is 1% of what is left, etc ...*

You can see that coin columns have different colors marking **reserve *(brown)* and **liquid** *(blue)*.

In the example above you can see that all coins with an index < 252 are reserve, while all coins >= 252 are liquid. This defines the **peg supply index** or in short **peg=252**.

**The peg system moves coins between reserve and liquid status, performing stabilization by expanding or contracting the total liquidity available on markets.**

Chapter 2: Data handling on exchange
-----

A typical example of data stored on an exchange:

![DataBefore](data-before-peg.png?raw=true "DataBefore")

After integrating the BitBay peg system, the new data structure will look like:

![DataAfter](data-after-peg.png?raw=true "DataAfter")

#### BAY and BAYR Trading Pairs

**After adapting the Dynamic Peg the coin is split into BAY (liquidity) and BAYR (reserve).**

This split results in two main trading pairs: BAY/BTC and BAYR/BTC. It is also possible to have liquid-reserve pair BAY-BAYR.

#### Pegcycle

The Dynamic Peg system is a cycle-based system. Each cycle has **200** blocks (reduced to 20 blocks in testnet). Cycles are aligned to 200 blocks, so the number of cycles can simply be calculated as **current_block_number/200**.

#### Peglevel

The **peglevel** field is hex data (64 bytes) calculated from the current **peg** index of the BAY blockchain and the exchange state of the liquidity. Typically an exchange would work on the **peg+1** level. The field has encoded information about the current cycle, peg, peg in next cycle and the liquidity requirement of the exchange.

**peglevel** is used for calculations of user balances.
**peglevel** is updated on cycle change and stays constant during the current cycle *(200 blocks)*.

#### Pegdata

The **pegdata** field is the compressed and base64-encoded information about the amounts of coins in all 1200 Baytoshi columns. The uncompressed data is an array of 1200 int64 values. These values can be plotted and visualized showing the coin fractions (as in the BitBay Qt wallet), or in charts and spreadsheets.

* **pegdata** of **Peg Balance information table/set** is used for balance computations.
* **pegdata** of **Peg Balance information table/set** is updated on **deposit**, **trade** and **withdraw**
* **pegdata** of **Exchange Peg information table/set** is used for tracking the exchanges cumulative coins
* **pegdata** of **Exchange Peg information table/set** is updated on **deposit** and **withdraw** to track the exchange totals

#### Pegpool

The **pegpool** field is the compressed and base64-encoded information about the liquidity pool state of the current **pegcycle**. It is used for liquidity pool trade computations. It is initialized at the beginning of the cycle and then used to update **pegbalances**.

#### Exchange Peg Information Table/Set

![DataAfter1](data-after-peg-peginfo.png?raw=true "DataAfter1")

This data represents the current **cycle**, the current **peglevel** and the cumulative **pegdata** of all BAY coins deposited to the exchange. There is also the possibility of additional **pegshift** pegdata which appears when Baytoshi columns of the exchange do not exactly match the Baytoshi columns of the transaction outputs of the bitbayd wallets. However, totals will always match.

Note that for BAY this is just one record as shown in the image.
*It is important for this data to be kept in the same database as user balances as it has to be in sync with exchange database transactions (deposits, trades and withdrawals) and has to follow the same database commits and rollbacks.*

#### Peg Cycles Information Table/Set

![DataAfter1a](data-after-peg-pegcycle.png?raw=true "DataAfter1a")

This data represents the history of **pegcycles**, the table/dataset is accumulating history information about peg cycles (every record is a copy of **Exchange Peg information table/set** at the end of the peg cycle). The purpose of the table is to keep track of liquidity pool data and to use for lazy updates of outdated pegbalances that have not been updated in the previous *pegcycle*.

*It is important for this data to be kept in the same database as user balances as it has to be in sync with exchange database transactions (deposits, trades and withdrawals) and has to follow the same database commits and rollbacks.*

#### Peg Balance Information Table/Set

![DataAfter2](data-after-peg-pegbalance.png?raw=true "DataAfter2")

This data represents information for the pair of balances (**BAY** and **BAYR**) for every owner of BAY on the exchange. The arrow indicates the reference to/from the standard **balance** table. The balances in the standard **balance** table are always calculated from the **peg balance** table.

It is impossible to manually adjust a users balance as it will be overwritten from **pegdata**. This provides a way to maintain the integrity of users balances as pairs of **reserve+liquidity**.

*It is important for this data to be kept in the same database as user balances as it has to be in sync with exchange database transactions (deposits, trades and withdrawals) and has to follow the same database commits and rollbacks.*

In the example above you can see that not all users have **cycle** in sync with the  **Exchange Peg information table/set**. When the **cycle** changes it is not necessary to force every user to update their reserve and liquid balances immediately. If a user is offline and does not have any SELL orders, the balances can be updated later. Therefore **Peg Balance table/set** implements **lazy** updates for user.

#### Example

**Alice** has her next data on cycle 330:

![DataAfterExample1](data-after-peg-example1.png?raw=true "DataAfterExample1")

Now the new cycle (next 200 blocks) is approached and the peg is changed from **51** to **53** for example. As the cycle changed, the exchange performed an RPC call to compute the new **peglevel** with *getpeglevel()*.

To update Alice’s balance, the exchange does an RPC call to bitbayd and sends the user **pegdata** and new **peglevel**. In response, bitabyd returns the new values for **total** of **BAY** and **total** of **BAYR** for Alice.

**Note that the total amount of BAY+BAYR does not change.**

![DataAfterExample2](data-after-peg-example2.png?raw=true "DataAfterExample2")

You can see that with the changed **peg=53** (peg increased), Alice now has less liquid coins and more reserve coins. The change also affected the **available** coins which is recalculated as **total - on_orders**.

*Note that Alice still has the same total amount of BAY+BAYR (10.00 in this example)*

Chapter 3: Keeping peginfo in sync
-----

### getpeginfo()

There is one main Json-RPC call that should be called regularly (e.g. every 10 seconds). The call is very light and just returns the current peg information from bitbayd:

Request:
```json
getpeginfo()
```

Response:
```json
{
"cycle" : 10213,
"peg" : 103,
"pegnext" : 103,
...
}
```

#### cycles

Indicates the current 200-blocks cycle (20-blocks in testnet). When this number changes the exchange has to perform:

1. compute new **peglevel** (via api call *getpeglevel()*)
2. check **SELL orders** to see if it requires appropriate updates (see Chapter 7)
3. register pending **deposits**
4. broadcast **withdrawals**

#### peg

Or in short, just **peg=103**. Indicates the current index of liquidity vs reserves in the system.

#### pegnext

This is the **peg** number in the next cycle.

### getpeglevel()

There is a second Json-RPC call that has to be invoked when **cycle change** is detected. The call requires the exchanges *pegdata* and *pegshift* to calculate peg levels that the exchange will be operating in this new cycle. For example:

Request:
```json
getpeglevel(<exchange_pegdata_base64>,<pegshift_pegdata_base64>,<previous_cycle>)
```

*Arguments:*
1. **exchange_pegdata_base64** to be taken from **Exchange Peg information table/set**
2. **pegshift_pegdata_base64** to be taken from **Exchange Peg information table/set**
3. **previous_cycle** to be taken from **Exchange Peg information table/set**

Response:
```json
{
"peglevel" : "380000000000000035003200a929000000000000000000000000000000000000",
"pegpool_pegdata" : "AgACAAAA...HAQA=",
...
}
```

#### peglevel

This is a binary hex-encoded dataset of 6 numbers, defining the level of the peg the exchange is operating on *(cycle,peg,pegnext,...)*. This data is saved in **Exchange Peg information table/set**.

#### pegpool

At begin of the cycle computation of liquidity information is performed and appropriate information is returned to be recorded into **Exchange Peg information table/set**. The **pegpool** information is then used for updating the user balances on the peg cycle change.

#### pegcycle

*Note there is an important step to copy current information from **Exchange Peg information table/set** as a new appended row/record to **Peg Cycles information table/set** before applying any updates to it.*

#### logic

![DataAfterExample3](data-after-peg-example3.png?raw=true "DataAfterExample3")

**Security and Performance:**

*Note that getpeginfo() **does not require private keys to operate**. The call requires only bitbayd to be in sync with the blockchain.*

*Note that getpeglevel() **does not require private keys to operate**. The call is stateless. If high performance is required then a C/C++ library or backend, language-based library can be implemented for fast computation of **peglevel**.*


Chapter 4: Deposits
-----

### Address

The generation of a deposit address can be done in the standard way using **getaccountaddress()**:

Request:
```json
getaccountaddress("alice")
```

Response:
```json
mxjcVYhfPP22uTchR2wzhWJPvgYN9hrSTX
```

### Detection

The detection of incoming deposits is performed in the standard way by periodically calling **listdeposits()** (which is doing exactly same as *listunspent()* but excluding transactions of exchange for withdraws and accounts maintenances) and comparing available *txouts* (format *hash:nout*) with database records to see which ones are already deposited or not.

*Note that **listdeposits()** has a parameter to return only outputs which have confirmations less than the specified parameter. This ensures it works efficiently and for an optimal amount of time.*

Request:
```json
listdeposits(1,200)
```

Response:
```json
[
{
"txid" : "0023336eefc1005a4d362d3a5c65ca07112e982743cf6bc2dfc39c87a10091a9",
"vout" : 2,
"address" : "mxjcVYhfPP22uTchR2wzhWJPvgYN9hrSTX",
"account" : "alice",
"scriptPubKey" : "76a914569eabb2495fa02713739f113657acf05cb6fb4b88ac",
"amount" : 5.00000000,
"reserve" : 4.00000000,
"liquidity" : 1.00000000,
"confirmations" : 4,
"spendable" : true
}
]
```

Logic:
![DepositExample1](data-after-peg-example4.png?raw=true "DepositExample1")

**Security:**

*Note that *listdeposits()* **does not require private keys to operate**. *listdeposits()* can be invoked for **bitbayd** which has deposit addresses only as **watch-only** added via **importaddress** - so the deposit detection system can run in a container without access to private keys. Also the set of addresses can be split over multiple running bitbayd to run distributed system of wallets.*

### Registering

When new output is detected then **registerdeposit()** should be initiated.

Request:
```json
registerdeposit(<txid:nout>,<balance_pegdata_base64>,<exchange_pegdata_base64>,<peglevel_hex>)
```

*Arguments:*

1. **txout** id (format *hash:nout*)
2. **balance_pegdata_base64** - to be taken from **Peg Balance information table/set**.
3. **exchange_pegdata_base64** - to be taken from **Exchange Peg information table/set**.
4. **peglevel_hex** - to be taken from **Exchange Peg information table/set**.

*Note that **pegdata** can be an empty string if the balance is zero (for the first call)*

The registration of the deposit needs to satisfy the following requirements:

1. Have 10 confirmations
2. Be aligned to the beginning of the next cycle

*Example 1:*

* Deposit transaction is mined at block 10030
* 10 confirmations at block 10040
* Registered at block 10200

*Example 2:*

* Deposit transaction is mined at block 10195
* 10 confirmations at block 10205
* Registered at block 10400

In **Example 2** at the beginning of the cycle (10200) the deposit transaction does not have enough confirmations so it is postponed for one more cycle.

*Response if not registered yet:*
```json
{
"deposited" : false,
"atblock" : 10400,
"status" : "Need to wait for registration at 10400 block"
}
```

*Response deposited:*
```json
{
"deposited" : true,
"status" : "Registered",
"atblock" : 10400,
"balance_value" : 1500000000,
"balance_liquid" : 1200000000,
"balance_reserve" : 300000000,
"balance_nchange" : 100000000,
"balance_pegdata" : "AAA....XXX",
"exchange_value" : 100000500000000,
"exchange_liquid" : 90000500000000,
"exchange_reserve" : 10000000000000,
"exchange_nchange" : 300000000,
"exchange_pegdata" : "AAA....XXX"
}
```

Logic:
![DepositExample2](data-after-peg-example5.png?raw=true "DepositExample2")

*Code and database transaction:*

*Note that registering a deposit requires different data from different tables to be read and written. It is important that all these operations are combined into one database transaction to keep all pieces of the exchange data in sync*

*Pseudocode:*
```c++
DatabaseBeginTransaction();
try {
	auto peglevel = ReadExchangePegInformation().peglevel;
	auto deposit_txout = ReadDepositInformationRow(deposit_id).txout;
	auto exchange_pegdata = ReadExchangePegInformation().pegdata;
	auto balance_pegdata = ReadBalancePegInformationRow("Alice").pegdata;
	auto response = DoJsonRPC("registerdeposit",
		deposit_txout,
		balance_pegdata,
		exchange_pegdata,
		peglevel
	);
	if (!response) {
		throw error(...);
	}
	UpdateDepositInformationRow(deposit_id, "status", "DEPOSITED");
	UpdateExchangePegInformation("pegdata", response.exchange_pegdata);
	UpdateBalancePegInformationRow("Alice", "pegdata", response.balance_pegdata);
	UpdateBalanceInformationRow("Alice", "BAY", "total", response.balance_liquid);
	UpdateBalanceInformationRow("Alice", "BAYR", "total", response.balance_reserve);
	CalculateAndUpdateAvailableBalance("Alice", "BAY");
	CalculateAndUpdateAvailableBalance("Alice", "BAYR");

	DatabaseCommitTransaction();
}
catch (...) {
	DatabaseRollbackTransaction();
	ReportError(...)
}
```

**Security:**

*Note that registerdeposit() **does not require private keys to operate**. The registerdeposit() can be invoked for bitbayd and only requires bitbayd to be in sync with the blockchain so the peg system has information about the requested txout.
No requirement for watch-only addresses.*


Chapter 5: Keep balances in sync with the peg
-----

When a user has to perform an *order* or *trade* operation with their BAY or BAYR balance, it is first mandatory to check **Peg Balance information table/set** - **cycle** is the same as **cycle** in the **Exchange Peg information table/set.**

![BalanceExample1](data-after-peg-example6.png?raw=true "BalanceExample1")

If they are not equal like the example above then the Json-RPC call **updatepegbalances** has to be performed first to sync the balances.

The call has to use and update liquidity pool information. The call **updatepegbalances()** can update pegbalance for one-step change of pegcycle. If user pegbalance is outdated then serie of **updatepegbalances()** has to be performed by information from **Peg Cycles information table/set**.

We recommend to do lazy updates of **pegbalance** of all users holders of BAY (include offline) so that **pegbalance** are kept up-to-date with current cycle and no long computations has to be performed in case of very outdated **pegbalance**.

Request:
```json
updatepegbalances(<balance_pegdata_base64>, <pegpool_pegdata_base64>, <peglevel_hex>)
```

Response:
```json
{
"balance_value" : 1500000000,
"balance_liquid" : 1200000000,
"balance_reserve" : 300000000,
"balance_nchange" : 100000000,
"balance_pegdata" : "AAA....XXX",
"pegpool_value" : 1500000000,
"pegpool_liquid" : 1200000000,
"pegpool_reserve" : 300000000,
"pegpool_nchange" : 100000000,
"pegpool_pegdata" : "AAA....XXX",
}
```

Updated *liquid* and *reserve* balances have to be written into the balance table and will then be operable for orders and trades.

Logic:

![BalanceExample2](data-after-peg-example7.png?raw=true "BalanceExample2")

**Code & database transaction:**

*Note that updating the balances requires data from different tables to be read and written. It is important that all these operations are packed into one database transaction to keep all of the exchange data in sync*

*Pseudocode for one-step pegbalance update to latest:*
```c++
DatabaseBeginTransaction();
try {
	auto exchange_cycle = ReadExchangePegInformation().pegcycle;
	auto balance_cycle = ReadBalancePegInformationRow("Alice").pegcycle;
	if (exchange_cycle == balance_cycle) {
		DatabaseRollbackTransaction();
		return; // cycle is same, no need to update
	}
	auto peglevel = ReadExchangePegInformation().peglevel;
	auto pegpool_pegdata = ReadExchangePegInformation().pegpool;
	auto balance_pegdata = ReadBalancePegInformationRow("Alice").pegdata;
	auto response = DoJsonRPC(
		"updatepegbalances",
		balance_pegdata,
		pegpool_pegdata,
		peglevel);
	if (!response) {
		throw error(...);
	}
	UpdateExchangePegInformation("pegpool", response.pegpool_pegdata);
	UpdateBalancePegInformationRow("Alice", "pegcycle", exchange_cycle);
    UpdatePegBalanceInformationRow("Alice", "pegdata", response.balance_pegdata);
	UpdateBalanceInformationRow("Alice", "BAY", "total", response.balance_liquid);
	UpdateBalanceInformationRow("Alice", "BAYR", "total", response.balance_reserve);
	CalculateAndUpdateAvailableBalance("Alice", "BAY");
	CalculateAndUpdateAvailableBalance("Alice", "BAYR");

	DatabaseCommitTransaction();
}
catch (...) {
	DatabaseRollbackTransaction();
	ReportError(...)
}
```

*Pseudocode for out-dated pegbalance update (more than one-step):*
```c++
auto pegcycles_rows = ReadPegCyclesRowsOrderedByCycleFrom(old_pegbalance_cycle);
for(auto pegcycle_row : pegcycles_rows) {
    DatabaseBeginTransaction();
    try {
        auto historic_cycle = pegcycle_row.pegcycle;
        auto balance_cycle = ReadBalancePegInformationRow("Alice").pegcycle;
        if (historic_cycle <= balance_cycle) {
            DatabaseRollbackTransaction();
            continue; // cycle is same, already updated, skip to next
        }
        auto peglevel = pegcycle_row.peglevel;
        auto pegpool_pegdata = pegcycle_row.pegpool;
        auto balance_pegdata = ReadBalancePegInformationRow("Alice").pegdata;
        auto response = DoJsonRPC(
            "updatepegbalances",
            balance_pegdata,
            pegpool_pegdata,
            peglevel);
        if (!response) {
            throw error(...);
        }
        UpdatePegCyclePoolInformation(pegcycle_row.id, response.pegpool_pegdata);
        UpdatePegBalanceInformationRow("Alice", "pegcycle", historic_cycle);
        UpdatePegBalanceInformationRow("Alice", "pegdata", response.balance_pegdata);
        DatabaseCommitTransaction();
    }
    catch (...) {
        DatabaseRollbackTransaction();
        ReportError(...);
        break; // or do multiple attempts to update
    }
}
```

**Security and Performance:**

*Note that **updatepegbalances does not require private keys to operate**. The call is stateless. It means that if high performance is required then the stack of multiple bitbayd’s can run to serve multiple casts for **updatepegbalances**.
A C/C++ library or backend language-based library can also be provided to perform the same computations as **updatepegbalances* to optimise RPC and the network.*


Chapter 6: Trades
-----

The trade process follows the exchange procedure of operating database transactions. The database transaction has to also include the Json-RPC call to bitbayd to update **pegdata** of the seller and buyer.

*Note: Before calling **moveliquid/movereserve** the existing balances should be in sync with the peg system (See previous chapter)*.

Request:
```json
moveliquid(<liquid>,<src_pegdata_base64>,<dst_pegdata_base64>,<pegsupply>)
movereserve(<reserve>,<src_pegdata_base64>,<dst_pegdata_base64>,<pegsupply>)
```

Response:
```json
{
"supply" : 53,
"src_value" : 1500000000,
"src_liquid" : 1200000000,
"src_reserve" : 300000000,
"src_nchange" : 100000000,
"src_pegdata" : "AAA....XXX",
"dst_value" : 1500000000,
"dst_liquid" : 1200000000,
"dst_reserve" : 300000000,
"dst_nchange" : 100000000,
"dst_pegdata" : "AAA....XXX",
}
```

Logic:

![TradeExample1](data-after-peg-example8.png?raw=true "TradeExample1")

**Code & database transaction:**

*Note that the trade process requires data from different database tables to be read and written. It is important that all these operations are packed into one **trade** database transaction to keep all of the exchange data in sync*

*Pseudocode:*
```c++
//first update peg balances, ok as separate database transactions
{
	auto exchange_cycle = ReadExchangePegInformationRow("BAY").cycle;
	auto src_balance_cycle = ReadBalancePegInformationRow("Alice").cycle;
	if (exchange_cycle != src_balance_cycle) {
    	// as separate database transaction
		UpdateUserPegBalance("Alice");
	}
	auto dst_balance_cycle = ReadBalancePegInformationRow("Bob").cycle;
	if (exchange_cycle != dst_balance_cycle) {
    	// as separate database transaction
		UpdateUserPegBalance("Bob");
	}
}
//now to perform trade database transaction
DatabaseBeginTransaction();
try {
	auto exchange_cycle = ReadExchangePegInformationRow("BAY").cycle;
	auto src_balance_cycle = ReadBalancePegInformationRow("Alice").cycle;
	if (exchange_cycle != src_balance_cycle) {
		UpdateUserPegBalance("Alice");
	}
	auto dst_balance_cycle = ReadBalancePegInformationRow("Bob").cycle;
	if (exchange_cycle != dst_balance_cycle) {
		UpdateUserPegBalance("Bob");
	}

	FillTradeOrders("Alice", "Bob", "BAY", 200000000, ...);

	auto src_balance_pegdata = ReadBalancePegInformationRow("Alice").pegdata;
	auto dst_balance_pegdata = ReadBalancePegInformationRow("Bob").pegdata;
	auto response = DoJsonRPC("moveliquid", 200000000, src_balance_pegdata, dst_balance_pegdata);
	if (!response) {
		throw error(...);
	}
	UpdateBalancePegInformationRow("Alice", "pegdata", response.src_pegdata);
	UpdateBalanceInformationRow("Alice", "BAY", "total", response.src_liquid);
	UpdateBalanceInformationRow("Alice", "BAYR", "total", response.src_reserve);

	UpdateBalancePegInformationRow("Bob", "pegdata", response.dst_pegdata);
	UpdateBalanceInformationRow("Bob", "BAY", "total", response.dst_liquid);
	UpdateBalanceInformationRow("Bob", "BAYR", "total", response.dst_reserve);

	CalculateAndUpdateAvailableBalance("Alice", "BAY");
	CalculateAndUpdateAvailableBalance("Alice", "BAYR");
	CalculateAndUpdateAvailableBalance("Bob", "BAY");
	CalculateAndUpdateAvailableBalance("Bob", "BAYR");

	AddTradeHistoryItem("Alice", "Bob", "BAY", 200000000, Datetime.Now(), ...)

	DatabaseCommitTransaction();
}
catch (...) {
	DatabaseRollbackTransaction();
	ReportError(...)
}
```

**Security and Performance:**

*Note that **moveliquid/movereserve does not require private keys to operate**. The call is stateless. It means that if high performance is required then the stack of multiple bitbayd’s can run to serve multiple casts for **moveliquid/movereserve**.
A C/C++ library or backend language-based library can also be provided to perform the same computations as **moveliquid/movereserve** to optimise RPC and the network.*

Chapter 7: Sell orders management
-----

As the peg changes over time, a user’s amount of liquid and reserve coins also change. It is therefore possible that a user’s balance of liquid or reserve coins is actually less than the amount on the exchange’s sell orders.

To solve this case during the peg changes:
1. Firstly, the **available amount** should decrease until it is non-zero (sell orders are not affected).
2. Secondly, when the available amount equals zero, all **sell orders** should decrease proportionally.

Example:

Alice’s balance and sell orders before the peg changes:

![OrdersExample1](data-after-peg-example9.png?raw=true "OrdersExample1")

After the peg changes and all available liquid coins became reserve (sell orders are not affected):

![OrdersExample2](data-after-peg-example10.png?raw=true "OrdersExample2")

After further peg changes there are no available coins and all of Alice's sell orders have now decreased proportionally to fit the liquid balance of **2.50**:

![OrdersExample3](data-after-peg-example11.png?raw=true "OrdersExample3")

*Note that if an order is partially filled (Order **id=3** in example above) then it should decrease only the coins that **remain**. *

**Order of updates:**

**It is important that sell orders are updated in order, starting from the minimum sell price as these orders are the first ones to be traded.** 
*The exchange does not need to pause trades until orders are updated with peg system.*

When the peg changes back, then the decreased orders can also be expanded. To implement this, there can be an additional column added to the orders table in the database (e.g. **initial_total**).

There is also the possible case when, due to the distribution of coins, the balance equals exactly zero (meaning all coins have either moved to *liquid* or *reserve*). For such cases, all orders of this user will also turn to zero. **We recommend not to close and delete** these orders, but turn them out of the common orders pool (ensuring they are *hidden* for other users to trade them), but kept visible in **my orders** where the user can cancel them or decide to keep them until the peg ‘relaxes’.

If the exchange also implements restrictions for **dust trades**, then these rules should be ‘dynamic’ for BAY coins as the available liquidity in the system can change drastically (200,000 times when the peg changes from 0 to maximum).

Chapter 8: Withdrawals
-----

#### Data structures

Withdrawals require some additional data to be handled by the database. When a user requests an amount of coins to be withdrawn, bitbayd performs the iteration over available unspent outputs *(txouts)* to rate them as inputs for the requested amounts to be withdrawn. The bitbayd then selects the best matched inputs to perform the same fractions *(Baytoshi)* to be sent to the user.

It is still possible to have a small ‘pegshift’ between what the exchange tracks as the users balance and what the user actually received. The exchange therefore keeps this cumulative **pegshift** and uses it on the cycle change to calculate the **peglevel**. This defines how peg balances are adjusted for liquid calculations.

Withdrawal transactions are computed with respect to the peg in the next cycle and will have plenty of time to be included in blocks.

![WithdrawExample1](data-after-peg-example12.png?raw=true "WithdrawExample1")

In the beginning the **pegshift** is empty, meaning that coins (Baytoshi) operable on the exchange exactly match the coins kept in the bitbayd wallets.

To prepare withdrawal transactions in advance and broadcast them later, the exchange requires a table/set in the database to keep information about spent inputs and the provided outputs:

![WithdrawExample3](data-after-peg-example14.png?raw=true "WithdrawExample3")

If the exchange uses just one bitbayd wallet then this information can be added to **Exchange Peg information table/set**. But if the exchange uses multiple wallets for security reasons (e.g. multiple hot wallets) then this information is **per bitbayd** and should be as separate entries/rows.

The withdrawals table is required to keep the signed **rawtransaction** and **for_cycle** when the raw transaction should be included in and be broadcasted.

![WithdrawExample4](data-after-peg-example15.png?raw=true "WithdrawExample4")

#### Timing

The withdrawal requests are collected during the the current cycle. During the withdrawal creation process the exchange makes the **prepareliquidwithdraw()/preparereservewithdraw()** Json-RPC request to bitbayd. 
As it is a resource- and time-consuming operation, bitbayd creates a transaction in advance, signs it and returns it to the exchange.

It is to broadcast all rawtrasaction in same routines which detect cycle change and prepare new **peglevel**. When new **peglevel** and **pegpool** are ready and written to database it is time to broadcast all pending **rawtransactions** which were prepared for this new **cycle**.


*Note that the transaction is not broadcast by bitbayd as it has been selected for the next cycle with a different peg.*

As a result, bitbayd returns:

- **hex-encoded transaction** ready for broadcast
- information about **transaction change** (not to be interpreted as deposit later)
- information about **consumed inputs** (because the transaction is not yet broadcast, another withdrawal transaction should not use the same inputs so as to avoid a conflict)
- information about **provided outputs** (these are **change** from transactions which can be now used as inputs by the next withdrawal transaction)

#### Fees

Please note that **prepareliquidwithdraw()/preparereservewithdraw()** get input parameter **amount_with_fee**. The system fee is fixed in current logic and set to **1M Baytoshi** which gives very high warranty for the coins to be included during next peg cycle. The exchange should include 1M fee in computations to show amount which user will receive. 

*amount_user_receive = amount_with_fee - 1M Baytoshi*

Same time the exchange may take own withdraw fees which to be deducted from user **pegbalance** after successfull completion of **prepareliquidwithdraw()/preparereservewithdraw()**. It is done with **movecoin()** from user account to exchange user account.

#### Processing

![WithdrawExample5](data-after-peg-example16.png?raw=true "WithdrawExample5")

**Code & database transaction:**

*Note that the withdraw process requires data from different tables to be read and written. It is important that these operations are packed into one **withdraw** database transaction to keep the exchange data in sync*

*Pseudocode:*
```c++
//first update peg balances, ok as separate database transactions
{
	auto exchange_cycle = ReadExchangePegInformation().pegcycle;
	auto balance_cycle = ReadBalancePegInformationRow("Alice").pegcycle;
	if (exchange_cycle != balance_cycle) {
		// as separate database transaction
		UpdateUserPegBalance("Alice");
	}

}
//now to perform withdraw database transaction
DatabaseBeginTransaction();
try {
	auto exchange_cycle = ReadExchangePegInformation().pegcycle;
	auto balance_cycle = ReadBalancePegInformationRow("Alice").pegcycle;
	if (exchange_cycle != balance_cycle) {
		UpdateUserPegBalance("Alice");
	}

	auto peglevel = ReadExchangePegInformation().peglevel;
	auto exchange_pegdata = ReadExchangePegInformation().pegdata;
	auto pegshift_pegdata = ReadExchangePegInformation().pegshift;
	auto balance_pegdata = ReadBalancePegInformationRow("Alice").pegdata;

	auto consumed_inputs = ReadPegWithdrawProcessingInformationRow("bitbayd1").consumed_inputs;
	auto provided_outputs = ReadPegWithdrawProcessingInformationRow("bitbayd1").provided_outputs;

	auto response = DoJsonRPC("prepareliquidwithdraw",
		balance_pegdata,
		exchange_pegdata,
		pegshift_pegdata,
		500000000, /*amount with fee*/
		"B6qyxfR8WPW7Mvrjkomv5Ep1SSNyBbbscB",
		peglevel,
		consumed_inputs,
		provided_outputs
	);
	if (!response) {
		throw error(...);
	}

	UpdateExchangePegInformation({
		"pegdata" : response.exchnage_pegdata,
		"pegshift" : response.pegshift_pegdata
	});
	UpdateBalancePegInformationRow("Alice", "pegdata", response.balance_pegdata);
	UpdateBalanceInformationRow("Alice", "BAY", "total", response.balance_liquid);
	UpdateBalanceInformationRow("Alice", "BAYR", "total", response.balance_reserve);
	CalculateAndUpdateAvailableBalance("Alice", "BAY");
	CalculateAndUpdateAvailableBalance("Alice", "BAYR");
	InsertWithdraw(
		"BAY",
		"Alice",
		500000000,
		"PENDING",
		response.txhash,
		exchange_cycle+1, /*just next cycle*/
		response.rawtx
	);
	UpdatePegWithdrawProcessing("bitbayd1", response.consumed_inputs, response.provided_outputs);
	for(auto change_item : response.change) {
		InsertPegWithdrawChange(change_item.txout, change_item.amount);
	}

	DatabaseCommitTransaction();
}
catch (...) {
	DatabaseRollbackTransaction();
	ReportError(...)
}
```

**Security:**

*Note that **prepareliquidwithdraw()/preparereservewithdraw() does require private keys to operate, to sign the transaction**. The call does not change bitbayd wallet however, as it only prepares the transaction based on available coins in the wallet and does not broadcast it.*

Appendix 1: Overview of calls
-----

### Technical info

| call               |bitbayd/library|          frequency      | need blockchain sync | need keys   |
|--------------------|---------------|-------------------------|----------------------|-------------|
|    getpeginfo()    |bitbayd        |     15 secs is ok       | yes                  | no          |
|    getpeglevel()   |library        |     once 200 blocks     | no                   | no          |
| getaccountaddress()|bitbayd        |     when create address | no                   | yes         |
|listdeposits()      |bitbayd        |     1 minute ok         | yes                  | yes, or can use watch-address |
|registerdeposit()   |bitbayd        |  when time to register  | yes                  | no |
|updatepegbalances() |library        |  one per user once 200 blocks | no                  | no |
|movecoins()         |library        |  cancel withdraw         | no                  | no |
|moveliquid()        |library        |  on trade                | no                  | no |
|movereserve()       |library        |  on trade                | no                  | no |
|prepareliquidwithdraw() |bitbayd    |  on withdraw             | yes                 | yes |
|preparereservewithdraw() |bitbayd    |  on withdraw             | yes                 | yes |

### Library for doing on-site peg computation is to include:

*Measured on one server core of Xeon E5-2670 0 @ 2.60GHz*
*Can run in parallel for fast performance*

| call               | cpu time  | performance |
|--------------------|-----------|------------|
|getpeglevel()       | 4.5 msecs | 220 cps |
|updatepegbalances() | 8.0 msecs | 125 cps |
|movecoins()         | 8.0 msecs | 125 cps |
|moveliquid()        | 8.0 msecs | 125 cps |
|movereserve()       | 8.0 msecs | 125 cps |

### Database actions by calls:

| call                    |peglevel|exchange pegdata|exchange pegpool|exchange pegshift|balance pegdata|
|-------------------------|--------|----------------|----------------|-----------------|---------------|
|getpeginfo()             |        |                |                |                 |               |
|getpeglevel()            |    W   |        R       |       W        |         R       |               |
|getaccountaddress()      |        |                |                |                 |               |
|listdeposits()           |        |                |                |                 |               |
|registerdeposit()        |    R   |       RW       |                |                 |       RW      |
|updatepegbalances()      |    R   |                |       RW       |                 |       RW      |
|movecoins()              |    R   |                |                |                 |       RW      |
|moveliquid()             |    R   |                |                |                 |       RW      |
|movereserve()            |    R   |                |                |                 |       RW      |
|prepareliquidwithdraw()  |    R   |       RW       |                |        RW       |       RW      |
|preparereservewithdraw() |    R   |       RW       |                |        RW       |       RW      |


### Database tables example (**mysql**)

```sql

# Dump of table pegcoin
# ------------------------------------------------------------

CREATE TABLE `pegcoin` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `cycle` int(11) NOT NULL DEFAULT 0,
  `peglevel` varchar(100) COLLATE utf8mb4_unicode_ci DEFAULT NULL,
  `pegdata` blob NOT NULL DEFAULT '',
  `pegpool` blob NOT NULL DEFAULT '',
  `pegshift` blob NOT NULL DEFAULT '',
  PRIMARY KEY (`id`),
ON DELETE CASCADE ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;


# Dump of table pegcycle
# ------------------------------------------------------------

CREATE TABLE `pegcycle` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `pegcoin_id` int(10) unsigned NOT NULL,
  `cycle` int(11) NOT NULL DEFAULT 0,
  `peglevel` varchar(100) COLLATE utf8mb4_unicode_ci DEFAULT NULL,
  `pegdata` blob NOT NULL DEFAULT '',
  `pegpool` blob NOT NULL DEFAULT '',
  `pegshift` blob NOT NULL DEFAULT '',
  PRIMARY KEY (`id`),
  KEY `pegcycle_pegcoin_id_foreign` (`pegcoin_id`),
  KEY `pegcycle_cycle_IDX` (`cycle`) USING BTREE,
  CONSTRAINT `pegcycle_pegcoin_id_foreign` FOREIGN KEY (`pegcoin_id`) REFERENCES `pegcoin` (`id`) ON DELETE CASCADE ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;


# Dump of table pegbalance
# ------------------------------------------------------------

CREATE TABLE `pegbalance` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `user_id` int(10) unsigned NOT NULL,
  `pegcoin_id` int(10) unsigned NOT NULL,
  `cycle` int(11) NOT NULL DEFAULT 0,
  `pegdata` blob NOT NULL DEFAULT '',
  PRIMARY KEY (`id`),
  KEY `pegbalance_user_id_foreign` (`user_id`),
  KEY `pegbalance_pegcoin_id_foreign` (`pegcoin_id`),
  CONSTRAINT `pegbalance_pegcoin_id_foreign` FOREIGN KEY (`pegcoin_id`) REFERENCES `pegcoin` (`id`) ON DELETE CASCADE ON UPDATE CASCADE,
  CONSTRAINT `pegbalance_user_id_foreign` FOREIGN KEY (`user_id`) REFERENCES `users` (`id`) ON DELETE CASCADE ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;


# Dump of table pegordershift
# ------------------------------------------------------------

CREATE TABLE `pegordershift` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `order_id` int(10) unsigned NOT NULL,
  `init_amount` bigint(20) unsigned NOT NULL DEFAULT 0 COMMENT 'initial trading amount',
  PRIMARY KEY (`id`),
  KEY `pegordershift_order_id_foreign` (`order_id`),
  CONSTRAINT `pegordershift_order_id_foreign` FOREIGN KEY (`order_id`) REFERENCES `orders` (`id`) ON DELETE CASCADE ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;


```

Appendix 2: Building bitbayd
-----

### Building on Ubuntu 

The builds on linux systems are done via **depends** (build system of bitcoin project). The usual procedure is next:

```sh
$ apt-get update
...
$ apt-get install -y tree make gcc g++ curl bzip2 automake autoconf libtool pkg-config
...
$ export ARCH="x86_64-pc-linux-gnu"
$ cd bitbay-core-peg-v1
$ cd depends 
$ make HOST=$ARCH NO_QT=1 -j4
... 
$ cd ..
$ ./autogen.sh
$ ./configure --prefix=$(pwd)/depends/$ARCH --with-gui=no --with-zlib --without-miniupnpc --enable-reduce-exports --disable-tests --disable-gui-tests --disable-bench
$ make -j2
``` 

### Run

```sh
$ cd src
$ mkdir -p /data/bay
$ ./bitbayd -testnet -datadir=/data/bay -staking=0 -port=21914 -server -rpcport=21915
```

### Dockerfile

```dockerfile
FROM ubuntu:xenial AS builder

RUN apt-get update
RUN apt-get install -y tree
RUN apt-get install -y make
RUN apt-get install -y gcc g++
RUN apt-get install -y curl
RUN apt-get install -y bzip2
RUN apt-get install -y automake
RUN apt-get install -y autoconf
RUN apt-get install -y libtool
RUN apt-get install -y pkg-config

COPY bitbay-core-peg-v1 /bay
RUN tree -L 1 /bay

ENV ARCH x86_64-pc-linux-gnu
WORKDIR /bay
RUN cd depends && make HOST=$ARCH NO_QT=1 -j4
RUN ./autogen.sh
RUN ./configure --prefix=$(pwd)/depends/$ARCH --with-gui=no --with-zlib --without-miniupnpc --enable-reduce-exports --disable-tests --disable-gui-tests
RUN make -j4
RUN ldd src/bitbayd

FROM ubuntu:xenial
COPY --from=builder /bay/src/bitbayd /usr/bin/
COPY --from=builder /bay/contrib/docker/baystart_testnet.sh /
RUN ldd /usr/bin/bitbayd

ENV RPC_USER bitbayrpc
ENV RPC_PASS pegforever
ENTRYPOINT ["/baystart_testnet.sh"]

```

### Run

```sh
$ mkdir -p /data/bay
$ docker run -d -e RPC_PASS=secret -v /data/bay:/data -p 21914:21914 -p 21915:21915 --name bitbayd1 bitbayd_image 
```
