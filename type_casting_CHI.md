# 背景

最近我一直在做《Computer System: A Programmer's Perspective》這本經典書籍中的習題。其中有一道題引起了我的注意，我決定為此寫一篇深入的分析。

這道題本身非常簡單：
```
// 假設 x 是任意 32 位元有符號整數，
// 下面這句條件判斷式是否永遠返回 true？如果不是，請舉出一個 x 的例子使其返回 false。
x == (int)(float)x
```

就這樣。

## 初始想法

我們知道在 C 語言中，整數和浮點數都是 32 位元。如果你熟悉 IEEE754 浮點數，你就會知道浮點數的小數部分只有 23 位元。這意味著，當我將 x 強制轉換為 float 時，第23位後的位元會被丟棄，因此會有一些精度損失。所以答案顯然是 FALSE。

那麼，構造一個使表達式結果為 false 的 x 值應該很容易。我只需要找一個具有超過 23 個有效位元的 x就可以了，比如0x7fffffff。

## 推演過程

```
int x = 0x7fffffff;

// x = 0b 0111 1111 1111 1111 1111 1111 1111 1111

// 下一步是將其轉換為浮點數

float f = (float)x;

// x = 1.11 1111 1111 1111 1111 1111 1111 1111 x 2^30
// x = 1.11 1111 1111 1111 1111 1111 1111 1111 x 2^(157-127)
// 指數部分 = 157 = 0b10011101
// 小數部分 = 11 1111 1111 1111 1111 1111 1 | 111 1111  (23位後被丟棄)
// 根據IEEE754 單精度浮點數格式：
// f = 0b 正負 | 指數部分 | 小數部分
// f = 0b 0 | 10011101 | 1111 1111 1111 1111 1111 111
// f = 0b 0100 1110 1111 1111 1111 1111 1111 1111
// f = 0x 4effffff (這是我認為 f 應有的值)
```

然而，當我用 `lldb` 測試時，`f` 的實際值是 `0x4f000000`。

你能看出我的錯誤嗎？

## 查看Assembly

實際上，在做這道題的時候，我不太清楚 CPU 內部是如何完成這類轉換的。比如，這一行程式碼編譯後會產生什麼？使用了什麼匯編指令？是否需要多步驟的位元移位、OR、AND等操作？

於是我寫了一個簡單C程式：
```
int main(){
	int x = 0x7fffffff;
	float f = (float)x;
	int x1 = (int)f;
}
```
首先設定 x 是最大的32位元有符號整數，然後轉換成浮點數，然後再轉換成有符號整數。

好吧，讓我們看看結果，首先將C語言編譯成Assembly匯編語言：

```
gcc trytry.c -S trytry.S
```

我在 Mac 的 ARM 晶片上運行 Ubuntu Docker 容器。因此編譯器輸出的是 ARM 匯編。

ARM 匯編代碼的一部分 (main 函數)：
```
main:
.LFB0:
	.cfi_startproc                  ; 不知道是什麼
	sub	sp, sp, #16                 ; 處理新的stack frame
	.cfi_def_cfa_offset 16          ; 不知道是什麼
	mov	w0, 2147483647              ; 將 0x7fffffff 移入 w0 寄存器
	str	w0, [sp, 4]                 ; str 應該代表 store，現在值被存儲在記憶體中
	ldr	s0, [sp, 4]                 ; ldr 應該是 load register？將值載入 s0 寄存器中
	scvtf	s0, s0                  ; scvtf，應該是有符號整數轉浮點數(signed int convert to float)！！這是我想知道的關鍵指令
	str	s0, [sp, 8]                 ; 將浮點數存入記憶體
	ldr	s0, [sp, 8]                 ; 將浮點數從記憶體加載到 s0 寄存器
	fcvtzs	s0, s0                  ; fcvtzs 應該是浮點數轉有符號整數，不確定 'z' 代表什麼。
	str	s0, [sp, 12]                ; 將結果存回記憶體
	mov	w0, 0                       ; 準備最後返回 0
	add	sp, sp, 16
	.cfi_def_cfa_offset 0
	ret
	.cfi_endproc
```

然後我查閱了 ARM 手冊 (https://developer.arm.com/documentation/100076/0100/A64-Instruction-Set-Reference/A64-Floating-point-Instructions/SCVTF--scalar--fixed-point-?lang=en)：

> Signed fixed-point Convert to Floating-point (scalar)。此指令將 32 位元或 64 位元通用暫存器中的有符號值轉換為浮點數值，使用 FPCR 指定的舍入模式，並將結果寫入 SIMD 和 FP 目標暫存器。

好吧... 它提到了舍入模式。除了丟棄多出位元之外還有其他的舍入模式嗎？？FPCR 是什麼？讓我也查一下...

FPCR (https://developer.arm.com/documentation/ddi0595/2020-12/AArch64-Registers/FPCR--Floating-point-Control-Register)

好吧... 這是一個 64 位元的配置，用於各種浮點數運算，例如如何在計算中處理 NaN？我們感興趣的是配置中的第 22-23 位元：

RMode，第 [23:22] 位元
舍入模式控制欄位。
| RMode | 含義                                 |
| ----- | ------------------------------------ |
| 0b00  | 舍入到最近（RN）模式。                |
| 0b01  | 向上舍入（RP）模式（向正無窮方向）。  |
| 0b10  | 向下舍入（RM）模式（向負無窮方向）。  |
| 0b11  | 向零舍入（RZ）模式。                  |
        
哦.. 所以有 4 種不同的舍入模式。在執行測試問題時，我使用的是哪一種？讓我用調試器檢查一下：

```
gcc trytry.c -g -o trytry
lldb trytry
breakpoint set --file trytry.c --line 4. # main() 中的第一行
run

register read fpcr
```

顯示結果如下：
```
(lldb) register read fpcr
    fpcr = 0x00000000
         = (AHP = 0, DN = 0, FZ = 0, RMode = 0, FZ16 = 0, IDE = 0, IXE = 0, UFE = 0, OFE = 0, DZE = 0, IOE = 0, NEP = 0, AH = 0, FIZ = 0)
```

好的！所以 RMode = 0，這意味著我們使用的是 RN 模式。

RN 模式如何處理有符號整數到浮點數的轉換？讓我再看一下手冊...
(https://developer.arm.com/documentation/ddi0487/maa/-Part-A-Arm-Architecture-Introduction-and-Overview/-Chapter-A1-Introduction-to-the-Arm-Architecture/-A1-5-Floating-point-support/-A1-5-8-Rounding?lang=en#babjdcahg4)

#### 舍入到最近（RN）模式
舍入到最近模式將浮點運算的精確結果舍入為目標格式中可表示的值，如下所示：

> 如果舍入前的值其絕對值太大，無法在輸出格式中表示，則舍入後的值為無窮大。舍入後值的符號與舍入前值的符號相同。

這不是我們的情況，float 的可表示範圍比有符號整數大得多，就算是最大的有符號整數都不會令浮點數發生溢出誤差。

> 如果舍入前的值其絕對值在輸出格式中可以表示，則結果計算如下：

這就是我們的情況！

> 如果原值與夾著原值的兩個最近的浮點數距離相等，則結果是最低有效位為偶數的那個數。

原來這就是 CSAPP 書中提到的「舍入到偶數」！

> 如果原值與夾著原值的兩個最近的浮點數距離不相等，則結果是離舍入前值最近的浮點數。

哦，我明白了，讓我重新計算一次！

```
x = 1.11 1111 1111 1111 1111 1111 1111 1111 x 2^30     // x 的精確值
x = 1.11 1111 1111 1111 1111 1111 1 | 111 1111 x 2^30  // 準確至小數點後23位

找出夾著原值的兩個浮點數中較接近原值的一方：
x = 10.00 0000 0000 0000 0000 0000 0 | 000 0000 x 2^30  (上界，離實際值更近)
x =  1.11 1111 1111 1111 1111 1111 1 | 111 1111 x 2^30  
x =  1.11 1111 1111 1111 1111 1111 1 | 000 0000 x 2^30  (下界)

因此，x 被向上舍入到 10 x 2^30 = 1 x 2^31 = 2^(158-127)！
// 指數部分 = 158 = 0b 10011110
// 小數部分 = 0

// f = 0 | 1001 1110 | 000 0000 0000 0000 0000 0000
// f = 0100 1111 0000 0000 0000 0000 0000 0000
// f = 0x4f000000
```

這與 f 的實際值完全一致！

## 第二個驚喜

然後這道題的剩餘部分是要將這個浮點數轉換回有符號整數。

但是... 剛才 x 被向上舍入了 1 變成 f，這意味著 f 超出了 32 位元二補數能表示的最大整數！

```
f = 2^31 		// f 的值
f = 0x80000000  // 超出了32位元有符號整數的最大值，直接轉換會發生溢出誤差！
```

如果直接轉換會發生什麼？讓我先在 ARM (Mac機) 上試試...

```
int x = (int)f;
// x = 0x7fffffff
```

嗯，結果變成了 32 位元有符號整數的最大可表示值... 為什麼？不是會發生溢出誤差嗎？這是偶然的嗎？讓我試試另一個更大的浮點數值，看看轉換成有符號整數會發生什麼事：

```
// x = 0x7fffffff
```

結果還是一樣！看起來 ARM CPU 在發生溢出時做了一些值限制（clamping），規定了溢出時只會以最大值作為結果。還記得 `fcvtzs` 指令嗎？讓我們查一下 ARM 的手冊：

FCVTZS 代表 Floating-point Convert to Signed integer, rounding toward Zero (scalar)。

Rounding towards zero 代表向零舍入模式，那什麼是向零舍入模式呢？繼續查看 ARM 手冊：

> 向零舍入模式將浮點運算的精確結果舍入為目標格式中可表示的值。結果是在輸出格式中，絕對值不大於舍入前值且最接近舍入前值的浮點數。

哦，原來如此！這就是為什麼它給我 32 位元有符號整數的最大可表示值，因為 0x7fffffff 是 32 位元有符號整數中最接近且不大於 0x80000000 的值！

這很有趣，我想知道在 x86_64 機器上 (一般Windows機器) 是否也是一樣的。因此，我安裝了交叉編譯器並查看 x86_64 版本的匯編：

```
x86_64-linux-gnu-gcc trytry.c -S -masm=intel trytry_x86_64.s
```

這是 x86_64 匯編代碼的一部分：
```
main:
.LFB0:
	.cfi_startproc                                   ; 何意味
	endbr64                                          ; 何意味
	push	rbp                                      ; 棧相關操作
	.cfi_def_cfa_offset 16							 ; 何意味
	.cfi_offset 6, -16								 ; 何意味
	mov	rbp, rsp                                     ; 棧相關操作
	.cfi_def_cfa_register 6     					 ; 何意味
	mov	DWORD PTR -12[rbp], 2147483647            	 ; 載入 x
	pxor	xmm0, xmm0                               ; 清除 xmm0 寄存器，有關xmm寄存器的知識也是很有趣的額外收穫，下次再講
	cvtsi2ss	xmm0, DWORD PTR -12[rbp]           	 ; cvt = convert，si = signed int，2 = to，ss = scalar single-precision (float)
	movss	DWORD PTR -8[rbp], xmm0                  ; 將浮點數載入記憶體
	movss	xmm0, DWORD PTR -8[rbp]                  ; 將浮點數重新載入暫存器
	cvttss2si	eax, xmm0                            ; cvt = convert，t = ??，ss = float，2 = to，si = signed int，t 是什麼？？
	mov	DWORD PTR -4[rbp], eax                    	 ; 將結果存入記憶體
	mov	eax, 0                                    	 ; 返回 0
	pop	rbp
	.cfi_def_cfa 7, 8
	ret
	.cfi_endproc
```

可以看得出 cvtsi2ss 和 cvttss2si 是我們現在關心的指令。

`cvtsi2ss`：作用與 ARM 的 `scvtf` 一樣，而且在 x86_64 中也有一個暫存器用於選擇4種舍入模式。

`cvttss2si`：中間的 `t` 到底是什麼？今次讓我在 x86_64 CPU 手冊中搜尋 (https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)。

cvttss2si 代表：
> Convert with truncation a scalar single precision floating-point value to a scalar double-word integer.

所以 `t` 代表截斷(丟棄多出位元)。當浮點數的值不能被有符號整數用二進制補碼數字精確表示，便會簡單地向零舍入。但這並沒有提供關於這條指令如何處理溢出情況的資訊！

讓我在手冊中搜尋這條cvttss2si指令：

> 當源暫存器中的值是 NaN (not a number)、無限，或超出可表示範圍時，返回整數 Indefinite。

我們正正就是第三種情況，f超出了整數可表示範圍，所以它會給我整數 *Indefinite*。

但這是什麼？讓我在手冊中搜尋Indefinite：

```
Integer Indefinite = 0x80000000
```

啊哈！這與 CSAPP 書中提到的內容一致：

> 從 float 或 double 轉換為 int 時，值會向零舍入。例如，1.999 會被轉換為 1，而 -1.999 會被轉換為 -1。此外，值可能會溢出。C 標準沒有指定這種情況下的固定結果。Intel 兼容的微處理器將 0x8000...000 指定為整數不定值。

一切都豁然開朗了！

至於我如何在搭載 ARM 晶片的 Mac 上編譯並運行 x86_64 架構的二進制檔案，我只是簡單運行了以下指令：
```
gcc -arch x86_64 -O0 -g trytry.c -o trytry_x86 // 編譯
lldb trytry_x86								   // 除錯
```

由於我的 Mac 有一層兼容層叫Rosetta 2，所以能夠直接運行 x86_64 的二進制機械碼。

## C 標準

從這個實驗中，我們可以得知 C 標準沒有明確說明當浮點數被強制轉換回有符號整數並發生溢出時應該返回什麼。在不同CPU架構的電腦上，同一個C程式會有不同的結果。

因此這條問題，如果我們假設 x = 0x7fffffff，那麼在x86_64的機器上，x==(int)(float)x 會是false，以在ARM的機器上會是true。

那麼我們從中學到了什麼教訓呢？那就是，永遠不要在代碼中做任何可能導致未定義行為(undefined behavior)的事情！

希望你覺得這篇文章很有趣！我是 Kelvin，一個正在學習編程的人。

# 延伸閱讀

1. IEEE754 浮點數： https://zh.wikipedia.org/zh-tw/IEEE_754
2. Computer System: A Programmer's Perspective: https://github.com/xubenji/csapp/blob/master/Computer%20Systems%20A%20Programmers%20Perspective%20(3rd).pdf
3. CPU指令集架構簡介:https://zh.wikipedia.org/zh-tw/%E6%8C%87%E4%BB%A4%E9%9B%86%E6%9E%B6%E6%A7%8B
4. ARM架構CPU手冊：https://developer.arm.com/documentation/
5. x86_64架構CPU手冊：https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html
