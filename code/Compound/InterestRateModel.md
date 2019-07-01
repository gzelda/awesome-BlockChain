# Compound-InterestRateModel合约分析

## [官网](https://compound.finance/)    [github地址](https://github.com/compound-finance/compound-money-market)

## 内部合约结构（五个内部合约）

- **ErrorReporter**
- **InterestRateModel**
- **CarefulMath**
- **Exponential**
- **StandardInterestRateModel**

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



## InterestRateModel合约分析

这是一个利率模型的抽象类，为了解耦，可以方便以后具体模型细节的更改。

```js
/**
- 函数目标: 基于总资产、总现金流、总借款额来得出当前放贷利率
- 接收参数解析
	-- asset: 合约（平台）内资产总额
	-- cash: 合约（平台）内总的现金流
	-- borrows: 合约（平台）内总的借款额
- 返回参数解析
	-- 第一个返回值代表的是方法执行成功或失败
	-- 第二个返回值代表的是当前放贷利率，返回结果为x*10^18，要得到真实利率需除以10^18
*/
function getSupplyRate(address asset, uint cash, uint borrows) public view returns (uint, uint);

/**
- 函数目标: 基于总资产、总现金流、总借款额来得出当前借款利率
- 接收参数解析
	-- asset: 合约（平台）内资产总额
	-- cash: 合约（平台）内总的现金流
	-- borrows: 合约（平台）内总的借款额
- 返回参数解析
	-- 第一个返回值代表的是方法执行成功或失败
	-- 第二个返回值代表的是当前借款利率，返回结果为x*10^18，要得到真实利率需除以10^18
*/
function getBorrowRate(address asset, uint cash, uint borrows) public view returns (uint, uint);

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

## StandardInterestRateModel合约分析

这个合约内的主合约，实现了标准利率模型。

```js
contract StandardInterestRateModel is Exponential {
	uint constant oneMinusSpreadBasisPoints = 9000;//用来计算的一个基础数值
    uint constant blocksPerYear = 2102400;//每年出块数量
    uint constant mantissaFivePercent = 5 * 10**16;//表达式表示的百分之5，实际上是0.05 * 10**18

    //利率模型合约内部的错误类
    enum IRError {
        NO_ERROR,//没有错误
        FAILED_TO_ADD_CASH_PLUS_BORROWS,//计算现金+借款失败
        FAILED_TO_GET_EXP,//获取表达式失败
        FAILED_TO_MUL_PRODUCT_TIMES_BORROW_RATE//
    }

    /**
    - 函数目标: 计算联合利用率，利用率计算公式为: (borrows/(cash+borrows))
    	-- 当借款总额为0时，那利用率就是0
    - 接收参数解析:
        -- cash: 合约内总现金流
        -- borrows: 总的借款额
    */
    function getUtilizationRate(uint cash, uint borrows) pure internal returns (IRError, Exp memory)
	
    /**
    - 函数目标: 计算利用率与年借贷利率
    	-- 计算利用率: getUtilizationRate函数
    	-- 年借款利率= 0.05 + (利用率 * 0.45) = 0.05 + ((borrows/(cash+borrows)) * 0.45)
    - 接收参数解析:
        -- cash: 合约内总现金流
        -- borrows: 总的借款额
    */
    function getUtilizationAndAnnualBorrowRate(uint cash, uint borrows) pure internal returns (IRError, Exp memory, Exp memory)
    
    /**
    - 函数目标: 计算放贷利率，计算公式为放贷率 = ( (利用率 * 9000 * 年借贷利率) / (10000 * 2102400) )
    	-- 详细计算公式:
    		放贷率 = ( ((borrows/(cash+borrows)) * 9000 * ( 0.05 + ((borrows/(cash+borrows)) * 0.45))) / (10000 * 2102400) )
    - 接收参数解析:
    	-- _asset: 合约内总资产额
        -- cash: 合约内总现金流
        -- borrows: 总的借款额
    */
    function getSupplyRate(address _asset, uint cash, uint borrows) public view returns (uint, uint)
    
    /**
    - 函数目标: 计算借贷利率，计算公式为借贷率= ( 年借款利率 / 2102400 )
    	-- 借贷率 = ( (0.05 + ((borrows/(cash+borrows)) * 0.45)) / 2102400 )
    - 接收参数解析:
    	-- _asset: 合约内总资产额
        -- cash: 合约内总现金流
        -- borrows: 总的借款额
    */
    function getBorrowRate(address _asset, uint cash, uint borrows) public view returns (uint, uint)

```





