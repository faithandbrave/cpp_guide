# 浮動小数点数型
## 組み込み浮動小数点数型
浮動小数点数はIEEE 754 (IEC 659) 規格で定められている。標準C++はターゲット環境によってはその規格通りではない可能性はあるが、メジャーなターゲット環境では規格通りだ。

IEEE 754に従った場合の、各浮動小数点数型は以下となる：

| 型 | ビット数 | 指数部ビット数 | 仮数部ビット数 | 説明 |
|----|----------|----------------|----------------|------|
| `float`  | 32 |  8 | 23 | 単精度 |
| `double` | 64 | 11 | 52 | 倍精度 |

`long double`型もあるが、いまのところ謎な存在なので考えなくてよい。

多くの用途には単精度の`float`がマッチする (ゲームも含む)。`double`はシミュレーション・科学計算といったより精度が必要な状況で使用する。

IEEE 754では16ビットの半精度浮動小数点数型も定義されるが、標準C++には導入されていない。コンパイラによっては`__fp16`型が拡張として定義される。

これらは2進数ベースの浮動小数点数型である。C20以降には10進数ベースの浮動小数点数型も定義される。


## 計算誤差
2進数ベースの浮動小数点数の演算では、計算誤差が発生する。10進数ベースの浮動小数点数を2進数ベースに変換する際に、ぴったり対応しない値が近似で表されることによって発生する現象である。

10進数ベースでは計算誤差は発生しない。

たとえば、以下のコードは表明失敗する。

```cpp
float f = 0.0f;
for (int i = 0; i < 10; ++i) {
    f += 0.1f;
}

assert(f == 1.0f);
```

変数`f`には加算時に計算誤差が蓄積されていくため、合計値は`1.0`と完全一致はしない値となる。

一方で、値を代入しただけでは計算誤差は発生しないため、`1.0`を代入した結果を`1.0`と等値比較した場合は`true`となる。

```cpp
float f = 1.0f;
assert(f == 1.0f); // OK
```

多くのフレームワークでは、浮動小数点数の比較をする際に計算誤差をどの程度許容するかをパラメータとして与えられるようになっている。

Google Testの場合、デフォルトの許容誤差を使用する`ASSERT_FLOAT_EQ(a, b)`マクロと、許容誤差をパラメータとして手動入力する`ASSERT_NEAR(a, b, error)`マクロが定義される。

- [浮動小数点数の比較 - Google Test](http://opencv.jp/googletestdocs/advancedguide.html#adv-floating-point-comparision)

Catch2の場合、浮動小数点数を比較するための`Approx`クラスが定義されており、これもまたデフォルトの許容誤差とパラメータとして手動入力する許容誤差がある。

- [Floating point comparisons](https://github.com/catchorg/Catch2/blob/devel/docs/assertions.md#floating-point-comparisons)


## 標準の丸め
```cpp
#include <iostream>

int main()
{
    int x = 2.5f;
    std::cout << x << std::endl; // 2

    int y = 1.5f;
    std::cout << y << std::endl; // 1

    int z = 1.7f;
    std::cout << z << std::endl; // 1
}
```

切り捨て (ゼロ方向への丸め)。


## NaNと軽量チェック
0.0の浮動小数点数を0.0で割った場合などにNaN (Not a Number, 非数) という特殊な値になるが、整数のゼロ割と違ってセグメンテーション違反でプログラムが異常終了したりはしない。そのため、計算中のどこかでNaNになってしまったが、その原因を突き止めるのがたいへん、という問題がある。

値を一つひとつ `if (std::isnan(f))` で見ていくこともできるが、巨大な浮動小数点数の配列を一つひとつチェックしていくのはとてもたいへんだ。

そんなときに軽量なNaNチェックとして、浮動小数点例外を使用できる。

NaNが発生した場合には`FE_INVALID`という浮動小数点例外が起きるので、それを検出すればよい。

```cpp
#include <cfenv>

float ar[N] = {…};
comupute(ar);

if (std::fetestexcept(FE_INVALID)) {
    std::cout << "NaNが発生した" << std::endl;
}
```


## 順序
浮動小数点数は半順序 (partial ordering) をもつ。

- 全順序 (strong ordering, total ordering) は、すべての値が大小比較できないといけない
- その下の弱順序 (weak ordering) はほかの値と同値である値が存在し、すべての値が大小比較できるわけではない場合。浮動小数点数には+0と-0があり、これらは同値で大小比較できない
- その下の半順序 (partial ordering) はさらに比較不能な値が存在する場合。浮動小数点数には比較不能な値としてNaNがある

標準C++のソート関係アルゴリズムや集合クラスなどは、型が弱順序以上をもつことを要求する。浮動小数点数型は弱順序をもたず半順序ではあるが、入力にNaNが含まれていなければ弱順序とみなせる。


## マニアクス
### 1.0の作り方
指数部が8ビットだとしたら、指数部0b0111'1111 (MSBだけ0でそれ以外を1にする)、仮数部は0。2.0はその指数部に+1。

```cpp
#include <iostream>
#include <cstdint>

class SingleFloat {
    std::uint32_t data_;
public:
    static constexpr int exponent_bits = 8;
    static constexpr int mantissa_bits = 23;
    static constexpr int bits = 32;

    static constexpr std::uint32_t max_exponent = (1 << exponent_bits) - 1;
    static constexpr std::uint32_t max_mantissa = (1 << mantissa_bits) - 1;

    std::uint32_t get_exponent() const noexcept {
        return (data_ >> mantissa_bits) & max_exponent;
    }

    SingleFloat& set_exponent(std::uint32_t e) {
        if (e > max_exponent) {
            throw std::invalid_argument("over max exponent");
        }
        data_ = (data_ & ~(max_exponent << mantissa_bits)) | (e << mantissa_bits);
        return *this;
    }

    float to_float() const {
        float result = 0;
        std::memcpy(&result, &data_, sizeof(float));
        return result;
    }
};

int main()
{
    SingleFloat one{};
    one.set_exponent(0b0111'1111);
    std::cout << one.to_float() << std::endl; // 1

    SingleFloat two{};
    two.set_exponent(0b0111'1111 + 1);
    std::cout << two.to_float() << std::endl; // 2
}
```

### 無限大
無限大は、指数部が最大である値。

```cpp
#include <iostream>
#include <cstdint>

class SingleFloat {
    std::uint32_t data_;
public:
    static constexpr int exponent_bits = 8;
    static constexpr int mantissa_bits = 23;
    static constexpr int bits = 32;

    static constexpr std::uint32_t max_exponent = (1 << exponent_bits) - 1;
    static constexpr std::uint32_t max_mantissa = (1 << mantissa_bits) - 1;

    std::uint32_t get_exponent() const noexcept {
        return (data_ >> mantissa_bits) & max_exponent;
    }

    SingleFloat& set_exponent(std::uint32_t e) {
        if (e > max_exponent) {
            throw std::invalid_argument("over max exponent");
        }
        data_ = (data_ & ~(max_exponent << mantissa_bits)) | (e << mantissa_bits);
        return *this;
    }

    float to_float() const {
        float result = 0;
        std::memcpy(&result, &data_, sizeof(float));
        return result;
    }

    bool is_inf() const {
        return get_exponent() == max_exponent;
    }
};

SingleFloat make_inf() {
    return SingleFloat{}.set_exponent(SingleFloat::max_exponent);
}

int main()
{
    std::cout << make_inf().to_float() << std::endl; // inf
}
```

## 便利ツール
- [Float Toy](http://evanw.github.io/float-toy/)
    - 内部表現と10進数値の相互変換・可視化

