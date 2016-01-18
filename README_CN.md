# JavaScript 高级表达式运算解析器
===============================

这个项目来源于 https://github.com/silentmatt/js-expression-eval

我重写了 [js-expression-eval](https://github.com/silentmatt/js-expression-eval) 的底层，以获得非常高的性能和很好的扩展性。

## 支持以下特性：

- 自定义运算符
- 单目、双目、后缀运算符
- 运算符重载
- 公式化简
- 公式迭代
- 将公式转换成JavaScript原生函数

## 使用帮助 

### 安装

```bash
npm install js-expression
```

### 使用

```javascript
var Parser = require('js-expression').Parser;

function Complex(r, i){
  this.r = r;
  this.i = i || 0;
}

Complex.prototype.toString = function(){
  return this.r + '+' + this.i + 'i';
}

var parser = new Parser();

parser.overload('+', Complex, function(a, b){
  return new Complex(a.r + b.r, a.i + b.i);
});

var c = parser.parse("a + b + 1");
var a = new Complex(1, 2);
var b = new Complex(3, 4);

//Complex { r: 5, i: 6 }
console.log(c.evaluate({a:a, b:b}));
```

## Parser 对象

**Parser( )**

构造函数，生成一个公式解析器对象。

**parse({expression: string})**

解析字符串，得到一个具体的表达式。

```javascript
var Parser = require('js-expression').Parser;

var parser = new Parser();
var expr = parser.parse('x ^ 2 + y ^ 2');
console.log(expr.evalute({x: 3, y: 4}));
```

**static parse**

parse 的静态版本。

```javascript
var Parser = require('js-expression').Parser;

var expr = Parser.parse('x ^ 2 + y ^ 2');
console.log(expr.evalute({x: 3, y: 4}));
```

**static evaluate({expression: string} [, {variables: object}])**

直接解析字符串得到公式并求值。

```javascript
var Parser = require('js-expression').Parser;
var result = Parser.evaluate('x ^ 2 + y ^ 2', {x:3, y:4});
```

**addOperator({operator: string}, {priority: number}, {handler: function})**

增加一个运算符。

```javascript
var parser = new Parser();

function Vector(x, y){
  this.x = x;
  this.y = y;
}

//vector cross
parser.addOperator('**', 3, function(a,b){
  return new Vector(a.x * b.y, -b.x * a.y);
});

var expr = parser.parse("a ** b");

//Vector { x: 4, y: -6 }
console.log(expr.evaluate({
  a: new Vector(1, 2),
  b: new Vector(3, 4)
}));
```

**addFunction({name: string}, {handler: function}[, {can_simplify: boolean} = true])**

增加一个函数。

```javascript
var parser = new Parser();

parser.addFunction('time', function(){
  return Date.now();
},false);


var expr = parser.parse("'abc?t='+time()");

console.log(expr.evaluate());

parser.addFunction('xor', function(a, b){
    return a ^ b;
});

var expr = parser.parse("xor(5, 7) + x + 1");

//((2+x)+1)
console.log(expr.simplify().toString());
```

**suffix operator**

后缀运算符：你可以在函数名前面加上`~`符号，表示定义的是一个“后缀运算符”。

```javascript
var parser = new Parser();

parser.addOperator('~%', 4, function(a){
  return a / 100;
});

var expr1 = parser.parse("((300% % 2)*10)!");

//3628800
console.log(expr1.evaluate());
```

**overload({operator: string}, {Class: constructor}, {handler: function})**

针对特定的数据类型重载一个运算符。**注意**重载的运算符触发条件为任意一个参数的类型满足该重载函数所匹配的类型，并且会自动将所有的参数类型转换为重载函数匹配的类型。

```javascript
var parser = new Parser();

function Vector(x, y){
  this.x = x;
  this.y = y;
}

//vector cross
parser.addOperator('**', 3, function(a,b){
  return new Vector(a.x * b.y, -b.x * a.y);
});

//vector add
parser.overload('+', Vector, function(a, b){
  return new Vector(a.x + b.x, a.y + b.y);
});

var expr = parser.parse("a ** b + c");

console.log(expr.toString()); //((a**b)+c)
console.log(expr.evaluate({ //Vector { x: 9, y: -7 }
  a: new Vector(1, 2),
  b: new Vector(3, 4),
  c: new Vector(5, -1),
}));
```

## Parser.Expression 对象

Parser.parse 方法返回一个 Expression 对象。

**evaluate([{variables: object}])**

用指定的变量值对表达式求值。

```javascript
var expr = Parser.parse("2 ^ x");

//8
expr.evaluate({ x: 3 });
```

**substitute({variable: string}, {expr: Expression, string, or number})**

用变量表达式对当前表达式进行迭代。

```javascript
var expr = Parser.parse("2 * x + 1");
//((2*x)+1)

expr.substitute("x", "4 * x");
//((2*(4*x))+1)

expr2.evaluate({ x: 3});
//25
```

**simplify({variables: object})**

对表达式进行化简。

```javascript
var expr = Parser.parse("x * (y * atan(1))").simplify({ y: 4 });
//(x*3.141592653589793)

var expr.evaluate({ x: 2 });
//6.283185307179586
```

**simplify_exclude_functions**

某些函数每次相同参数调用时返回值不一定相同（比如random），这类函数不能用`simplify`化简。我们可以在addFunction函数中设置最后一个参数为false来避免函数被`simplify`化简。默认`random`函数不可以化简。

```javascript
var expr = Parser.parse("1 + random()").simplify();
//(1+random())
```

**variables([{include_functions: boolean}])**

```javascript
//Get an array of the unbound variables in the expression.

var expr = Parser.parse("x * (y * atan(1))");
//(x*(y*atan(1)))

expr.variables();
//x,y

expr.variables(true);
//x,y,atan

expr.simplify({ y: 4 }).variables();
//x
```

**toString()**

将表达式转成字符串。

**toJSFunction({parameters: Array} [, {variables: object}])**

将表达式转化为JavaScript函数。

```javascript
var expr = Parser.parse("x ^ 2 + y ^ 2 + 1");
var func1 = expr.toJSFunction(['x', 'y']);
var func2 = expr.toJSFunction(['x'], {y: 2});

func1(1, 1);
//3

func2(2);
//9
```

## 默认支持的运算符

**单目运算符**

	"-": neg,
	"+": positive,

**后缀运算符**

	"!": fac //阶乘

**双目运算符**

	"+": add,
	"-": sub,
	"*": mul,
	"/": div,
	"%": mod,
	"^": Math.pow,
	"||": concat,
	"==": equal,
	"!=": notEqual,
	">": greaterThan,
	"<": lessThan,
	">=": greaterThanEqual,
	"<=": lessThanEqual,
	"and": andOperator,
	"or": orOperator

**特殊运算符**

	",": comma,

**内置函数**

	"random": random,
	"fac": fac,
	"min": Math.min,
	"max": Math.max,
	"hypot": hypot,
	"pyt": hypot, // backward compat
	"pow": Math.pow,
	"atan2": Math.atan2,
	"cond": condition,

	"sin": Math.sin,
	"cos": Math.cos,
	"tan": Math.tan,
	"asin": Math.asin,
	"acos": Math.acos,
	"atan": Math.atan,
	"sinh": sinh,
	"cosh": cosh,
	"tanh": tanh,
	"asinh": asinh,
	"acosh": acosh,
	"atanh": atanh,
	"sqrt": Math.sqrt,
	"log": Math.log,
	"lg" : log10,
	"log10" : log10,
	"abs": Math.abs,
	"ceil": Math.ceil,
	"floor": Math.floor,
	"round": Math.round,
	"trunc": trunc,
	"exp": Math.exp

## 测试

执行测试用例：

1. [Install NodeJS](https://github.com/nodejs/node)
2. Install Mocha `npm install -g mocha`
3. Install Chai `npm install chai`
4. Execute `mocha`
