# Backend IR

## IR vs AST

AST（abstract syntax tree）的主要目的是表达语言源码，但在进行optimization和code gen的时候就不一定合适。

* AST面向的是用户，因此存在冗余大量语法糖，这对于用户是有益的，但对于Simulator来说无疑会增加优化和codegen的难度。

```verilog
module automatic top;
    function hello;
        $display("hello, world!");
    endfunction
    initial hello();
endmodule
```

在上面的例子中，修饰top的automatic只是一个设置module中task/function默认生命周期的语法糖。在AST中通常会保留以反应实际的源码，但实际在function的生命周期确定之后，我们就不再需要，因此也没有必要在后续的优化和cg时保留。

* Verilog中存在大量等价的表示，如果都保留显然会增加Simulator对其进行优化处理的难度。

```verilog
wire o;
not (o, i);         // 表示1
assign o = ~i;      // 表示2   

logic o;
always_comb o = ~i; // 表示3
```

上面例子中的3种表示都描述了一个not gate，可以用一种统一的方式表示。从另一方面讲，我们实际需要比AST抽象层级更高的表示。

因此将AST转化为抽象层级更高，更面向优化和cg的表示（即IR），是Simulator中不可避免的环节。

## Static & dynamic Verilog

通常Verilog可以分成static和dynamic两个部分:

* Static部分包括整个module / program / interface等构成的instance tree，以及其中的非automatic task / function。其特点是生命周期贯穿整个simulation始终。
* Dynamic部分组要是class以及所有automatic task/function。

区分static和dynamic部分的主要原因在于，static部分是Verilog区别于其它语言的主要特征，换言之也是Verilog中需要额外优化的部分。而其dynamic部分基本可以用其它语言比如C++替换，因此可以较方便的通过翻译或者其它形式实现。

## Static IR

所谓Static IR即对Verilog的static部分的抽象，在我们的模型中static IR只包括两个部分。

1. 定义每个Event逻辑的Process。
2. 定义Event传播的连接关系 (Driver / Reader关系)，这样的连接关系通常可以用网表netlist来表示。

如果我们Process视为顶点(vertex)，每个Process又会写若干信号，别的Process又会对这些信号敏感，这样Process之间就有了Driver / Reader关系，也就构成了边(edge)。这样由vertex和edge构成的graph，也就是需要仿真的design。

需要注意但经常被忽略的一点是，Process和Netlist是可以解耦的，将两者解耦可以避免在处理其一时引入过多不必要的依赖，这对于一个高效的Simulator是极端重要的。

### Netlist

Netlist有两个不同阶段，Folded阶段和Flatten阶段。

#### Folded阶段

在此阶段，module / instance / port仍然存在，因此不同instance之间可以共享同一份定义。此时的netlist可以用Verilog structure code来表示，例如：

```verilog
module top;
    wire q, clk;
    logic d;
    
    fd fd1(.q(q), .clk(clk), .d(d));
    clockgen cg1(.clk(clk));
    
    initial begin
        $monitor("#4t: d = %b, clk = %b, q = %b", $time, d, clk, q);
        #7 d = 1'b0;
    end
endmodule

module fd(output q, input clk, input d);
    always_ff @ (posedge clk)
        q <= d;
endmodule

module clockgen(output clk);
    logic clk = 0;
    always #5 clk = ~clk;
endmodule
```

netlist表示如下:

```verilog
module top;
    logic q, clk, d;
    fd fd1(.q(q), .clk(clk), .d(d));
    clockgen cg1(.clk(clk));
    
    process_init initial1 (.o(d), .i0(clk), .i1(q));
endmodule

module fd(q, clk, d);
    logic output q;
    logic input clk;
    logic input d;
    process_always_ff always_ff1(.o(o), i0(clk), i1(d));
endmodule

module clockgen(clk);
    logic output clk;
    process_always always1(.o(clk));
endmodule
```
