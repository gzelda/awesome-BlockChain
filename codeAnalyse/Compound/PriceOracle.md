# Compound-PriceOracle合约分析
## [官网](https://compound.finance/)    [github地址](https://github.com/compound-finance/compound-money-market)
## 内部合约结构（五个内部合约）

- **ErrorReporter**
- **DSValue**
- **CarefulMath**
- **Exponential**
- **PriceOracle**

## ErrorReporter合约分析

这是一个错误报告的类，将平台主合约MoneyMarket可能发生的错误进行了归类，并做详细记录

```js
/**
- 这是一个event类型用来记录错误事件
- 接收参数解析
	-- error: 合约内枚举类Error
	-- info: 合约内枚举类FailureInfo
	-- detail: 特定的错误代码，同样也是枚举类Error中的OPAQUE_ERROR字段
*/
event Failure(uint error, uint info, uint detail);

/**
- 函数目标: 开发人员使用，在出现已知错误时使用此方法来报告错误
- 接收参数解析
	-- err: 合约内枚举类Error
	-- info: 合约内枚举类FailureInfo
*/
function fail(Error err, FailureInfo info) internal returns (uint) {
        emit Failure(uint(err), uint(info), 0);

        return uint(err);
}

/**
- 函数目标: 开发人员使用，用来报告不能返回的错误
- 接收参数解析:
	-- info: 合约内的枚举类FailureInfo
	-- opaqueError: 特定的错误代码
- 返回参数解析:
	-- 枚举类Error当中的OPAQUE_ERROR字段（特定错误码）
*/
function failOpaque(FailureInfo info, uint opaqueError) internal returns (uint) {
        emit Failure(uint(Error.OPAQUE_ERROR), uint(info), opaqueError);

        return uint(Error.OPAQUE_ERROR);
}
```

## DSValue合约分析

这是一个实现了数值读取的类的合约，只提供两个数据的读取的抽象方法，同样是为了解耦合，后续方便合约升级时逻辑的更改。

```js
function peek() public view returns (bytes32, bool);

function read() public view returns (bytes32);
```

## CarefulMath合约分析

内部实现的安全的一些数学计算，继承了ErrorReporter合约，在出现错误时能够报告错误原因

```
contract CarefulMath is ErrorReporter
```

具体函数实现，因为金融市场跟钱挂钩，对于数的处理得特别小心，防止出现数据上的错误，尤其是加减乘除都得做成安全的防止溢出或者不合规范的计算。

```js
//乘
function mul(uint a, uint b) internal pure returns (Error, uint) 
//除
function div(uint a, uint b) internal pure returns (Error, uint) 
//减
function sub(uint a, uint b) internal pure returns (Error, uint) 
//加
function add(uint a, uint b) internal pure returns (Error, uint) 
//先加后减
function addThenSub(uint a, uint b, uint c) internal pure returns (Error, uint) 
```

## Exponential合约分析

这是一个平台内真正意义上实现了安全计算的合约，同时也是提高了计算的精度，因为任何计算的返回结果都是预先*10^18。

```js
contract Exponential is ErrorReporter, CarefulMath
```

```js
/**
- 内部定义了一个表达式类，将真正的计算结果进行了封装
*/
struct Exp {
        uint mantissa;
}

/**
- 函数目标: 将分子和分母的除法，转换为一个精确又合理的表达式形式(分子先乘10^18再做除法)
- 接收参数解析:
	-- num: 分子
	-- denom: 分母
- 返回参数解析:
	-- 枚举类Error
	-- 表达式类Exp
*/
function getExp(uint num, uint denom) pure internal returns (Error, Exp memory)

/**
- 函数目标: 将两个表达式相加
- 接收参数解析:
	-- a: 表达式1
	-- b: 表达式2
- 返回参数解析:
	-- 枚举类Error
	-- 表达式类Exp
*/
function addExp(Exp memory a, Exp memory b) pure internal returns (Error, Exp memory)

/**
- 函数目标: 将两个表达式相减
- 接收参数解析:
	-- a: 表达式1
	-- b: 表达式2
- 返回参数解析:
	-- 枚举类Error
	-- 表达式类Exp
*/
function subExp(Exp memory a, Exp memory b) pure internal returns (Error, Exp memory)

/**
- 函数目标: 将表达式*一个数值来获得一个新的表达式
- 接收参数解析:
	-- a: 表达式
	-- scalar: 数值
- 返回参数解析:
	-- 枚举类Error
	-- 表达式类Exp
*/
function mulScalar(Exp memory a, uint scalar) pure internal returns (Error, Exp memory)

/**
- 函数目标: 将表达式/一个数值来获得一个新的表达式
- 接收参数解析:
	-- a: 表达式
	-- scalar: 数值
- 返回参数解析:
	-- 枚举类Error
	-- 表达式类Exp
*/
function divScalar(Exp memory a, uint scalar) pure internal returns (Error, Exp memory)

/**
- 函数目标: 将一个数值/一个表达式来返回一个新的表达式
- 接收参数解析:
	-- scalar: 数值（除数）
	-- divisor: 表达式（被除数）
- 返回参数解析:
	-- 枚举类Error
	-- 表达式类Exp
*/
function divScalarByExp(uint scalar, Exp divisor) pure internal returns (Error, Exp memory)

/**
- 函数目标: 将两个表达式相乘
- 接收参数解析:
	-- a: 表达式1
	-- b: 表达式2
- 返回参数解析:
	-- 枚举类Error
	-- 表达式类Exp
*/
function mulExp(Exp memory a, Exp memory b) pure internal returns (Error, Exp memory)

/**
- 函数目标: 将两个表达式相除
- 接收参数解析:
	-- a: 表达式1
	-- b: 表达式2
- 返回参数解析:
	-- 枚举类Error
	-- 表达式类Exp
*/
function divExp(Exp memory a, Exp memory b) pure internal returns (Error, Exp memory)

/**
- 函数目标: 截断一个表达式为一个整数值，做法就是将表达式类内的数值除以10^18
- 接收参数解析:
	-- exp: 表达式
- 返回参数解析:
	-- 截断之后的整数值
*/
function truncate(Exp memory exp) pure internal returns (uint)

/**
- 函数目标: 比较第一个表达式是否比第二个表达式小（不含等于）
- 接收参数解析:
	-- left: 表达式1
	-- right: 表达式2
- 返回参数解析:
	-- true为小，false为不小于
*/
function lessThanExp(Exp memory left, Exp memory right) pure internal returns (bool)

/**
- 函数目标: 比较第一个表达式是否比第二个表达式小（含等于）
- 接收参数解析:
	-- left: 表达式1
	-- right: 表达式2
- 返回参数解析:
	-- true为小于等于，false为大于
*/
function lessThanOrEqualExp(Exp memory left, Exp memory right) pure internal returns (bool)

/**
- 函数目标: 判断这个表达式的值是否为0
- 接收参数解析:
	-- value: 表达式
- 返回参数解析:
	-- true为0，false为非0
*/
function isZeroExp(Exp memory value) pure internal returns (bool)
```

## PriceOracle合约分析

这是这个合约类中的主合约，主要是用来提供各个ERC-20代币的价格信息的。

```js
contract PriceOracle is Exponential
```

我们先来看所有上链的数据

```js
	//这是一个标志，标识这个合约是否被停用
	bool public paused;
	//这是阶段区块数，阶段为1小时，计算方法为: 60 * 60 / 15 
	//15秒为出一个块的时间
    uint public constant numBlocksPerPeriod = 240; 
	//价格最大的波动百分比，其实代表的是0.1
    uint public constant maxSwingMantissa = (10 ** 17);
	/**
	- 将地址上的金额总数映射为可用转换成实际的价值
		-- eth:asset类型 如 1eth = 100USD
		-- asset:eth类型 如 1USD = 1/100 eth
	*/
	mapping(address => DSValue) public readers;
	/**
	- 将资产地址来转换为Eth-wei类型，即用表达式来表达这个价格数值
		-- 例如: 1eth = 1 * 10**18
	*/
	mapping(address => Exp) public _assetPrices;
	//锚定价格的人
	address public anchorAdmin;
	//候选锚定价格的人
    address public pendingAnchorAdmin;
    //价格提供人
    address public poster;
	//锚定价格与新价格之间波动最大的百分比（表达式形式）
    Exp public maxSwing;
    //将地址转换为Anchor类的映射，即将资产映射到类中
    mapping(address => Anchor) public anchors;
	//候选人所拥有的资产
    mapping(address => uint) public pendingAnchors;
```

接下来看合约内的结构体

```js
//结构体：锚定
struct Anchor {
    	//周期，计算方式为 block.number / numBlocksPerPeriod 向下取整再 +1
        uint period;
        //价格转换为ETH，再乘10^18，即转换为Eth-wei
        uint priceMantissa;
}

//结构体：设置合约内存储的价格变量
struct SetPriceLocalVars {
        Exp price;//价格
        Exp swing;//波动范围
        Exp anchorPrice;//锚定价格
        uint anchorPeriod;//锚定时间段
        uint currentPeriod;//当前时间段
        bool priceCapped;//价格是否封顶
        uint cappingAnchorPriceMantissa;//封顶的锚定价格
        uint pendingAnchorMantissa;//候选的锚定价格尾数(10^18)
}
```

再来看合约初始化函数

```js
/**
- 函数目标: 初始化锚定价格的人、价格提供者
*/
constructor(address _poster, address addr0, address reader0, address addr1, address reader1) public {
    	//初始化锚定价格的人的地址
        anchorAdmin = msg.sender;
    	//初始化价格的提供者的地址
        poster = _poster;
    	//初始化最大的价格波动范围（当前价格与锚定价格）
        maxSwing = Exp({mantissa : maxSwingMantissa});

        //确保资产为0或者地址不相同
        assert(addr0 == address(0) || (addr0 != addr1));

        if (addr0 != address(0)) {
            assert(reader0 != address(0));
            readers[addr0] = DSValue(reader0);
        } else {
            assert(reader0 == address(0));
        }

        if (addr1 != address(0)) {
            assert(reader1 != address(0));
            readers[addr1] = DSValue(reader1);
        } else {
            assert(reader1 == address(0));
        }
}
```

剩下的是合约内的具体函数解析

```js
/**
- 函数目标: 更改某个ERC20代币目前的价格
	-- 必须是由锚定价格的人来调用
- 接收参数解析
	-- asset: 某种代币的地址
	-- newScaledPrice: 更改后的价格
*/
function _setPendingAnchor(address asset, uint newScaledPrice) public returns (uint)

/**
- 函数目标: 更改是否停用当前合约
	-- 必须是由锚定价格的人来调用
- 接收参数解析
	-- requestedState: 是否停用当前合约,true或false
*/
function _setPaused(bool requestedState) public returns (uint)

/**
- 函数目标: 更改候选锚定价格的人
	-- 必须是由锚定价格的人来调用
- 接收参数解析
	-- newPendingAnchorAdmin: 新的候选人的地址
*/
function _setPendingAnchorAdmin(address newPendingAnchorAdmin) public returns (uint)

/**
- 函数目标: 替换当前锚定价格的人，将候选锚定价格的人顶掉原来的锚定价格的人，并清除候选人
	-- 必须是由候选锚定价格的人来调用
*/
function _acceptAnchorAdmin() public returns (uint)

/**
- 函数目标: 获取某个代币目前的价格
	-- 如果合约已经停用，即paused为true，则返回0
	-- 如果代币价格可读，则返回对应价格，否则返回0
*/
function assetPrices(address asset) public view returns (uint)

/**
- 函数目标: 获取某个代币目前的价格，只是调用了一下assetPrices函数而已
	-- 这个方法才是获取价格的入口
*/
function getPrice(address asset) public view returns (uint)

/**
- 函数目标: 修改某个代币存储在合约内的价格
- 接收参数解析
	-- asset:代币地址
	-- priceMantissa:代币价格
*/
function setPriceStorageInternal(address asset, uint256 priceMantissa) internal {
        _assetPrices[asset] = Exp({mantissa: priceMantissa});
}

/**
- 函数目标: 计算改变后的价格与当前锚定价格的波动
	-- 计算方式: abs(price - anchorPrice) / anchorPrice
- 接收参数解析
	-- anchorPrice: 当前锚定价格
	-- price: 更新后价格
*/
function calculateSwing(Exp memory anchorPrice, Exp memory price) pure internal returns (Error, Exp memory)

/**
- 函数目标: 计算价格在波动范围内的上下限
	-- max = anchorPrice * (1 + maxSwing)
	-- min = anchorPrice * (1 - maxSwing)
	-- 保证代币的价格不超过上限或者下限，如果超过就返回上限或下限的价格值
- 接收参数解析
	-- anchorPrice: 锚定价格
	-- price: 更新后价格
*/
function capToMax(Exp memory anchorPrice, Exp memory price) view internal returns (Error, bool, Exp memory)

/**
- 函数目标: 进行价格锚定的操作，即具体更新代币价格的操作
- 接收参数解析
	-- asset: 代币地址
	-- requestedPriceMantissa: 代币价格
*/
function setPriceInternal(address asset, uint requestedPriceMantissa) internal returns (uint)

/**
- 函数目标: 进行价格更新的操作，这是更新价格的入口函数
	-- 只能由poster来调用这个函数
	-- 内部就是调用了setPriceInternal函数
*/
function setPrice(address asset, uint requestedPriceMantissa) public returns (uint)

/**
- 函数目标: 更新多个代币的价格
	-- 只能由poster来调用这个函数
- 接收参数解析
	-- assets: 多个代币地址
	-- requestedPriceMantissas: 多个代币分别对应的价格
*/
function setPrices(address[] assets, uint[] requestedPriceMantissas) public returns (uint[] memory)
```

