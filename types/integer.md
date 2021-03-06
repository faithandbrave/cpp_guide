# 整数型
## 組み込み整数型

| 型 | 説明 | 比較カテゴリ |
|----|------|--------------|
| `bool`  | 真理値型。値として`false`および`true`のみをもつ | 全順序 |
| `char`  | 文字型       | 全順序 |
| `short` | 小さい整数型 | 全順序 |
| `int`   | デフォルトの整数型 | 全順序 |
| `long (int)` | 大きい整数型 | 全順序 |
| `long long (int)` | さらに大きい整数型。64ビット以上 | 全順序 |

`bool`以外の型には`unsigned`を付加することで符号なし指定ができる。デフォルトは`signed` (符号付き)。ただし、`signed char`と`char`は別な型と見なされる。

### bool
真理値型。`false` (偽) と `true` (真) の二値のみを表現できる。値域としては1ビットでも表現できるが、ビットパディングの問題から8ビット (1バイト) で実装される。コンパイラのオプションとして32ビット (4バイト) にすることもできる。

### 整数型のビット数
規格としては、`int`は16ビット以上、`long`は32ビット以上、`long long`は64ビット以上であることと規定される。

これは2020年の現代的なアーキテクチャでは現実として以下のようになっている：

| 型 | ビット数 | 値の範囲 |
|----|----------|----------|
| `char`      | 8  | 符号付きは`[-128, 127]`、符号なしは`[0, 255]` |
| `short`     | 16 | 符号付きは`[-32768, 32767]`、符号なしは`[0, 65535]` |
| `int`       | 32 | 符号付きは`[-2147483648, 2147483647]`、符号なしは`[0, 4294967295]` |
| `long`      | ポインタサイズのビット数 | |
| `long long` | 64 | 符号付きは`[-9223372036854775808, 9223372036854775807]`、符号なしは`[0, 18446744073709551615]` |

「`int`が16ビットだったら…」というのは、メジャーなターゲット環境では考える必要はなく、32ビットと考えてよい。16ビットだとしたら、現代のプログラムとしては値の範囲が狭すぎて、`int`をデフォルトで使うことは困難であろう。

`long`はポインタサイズのビット数となっており、ILP32環境では32ビット、LP64環境では64ビットとなる。64ビットCPUだから`long`型が64ビットになるのではなく、64ビットCPUを搭載したマシンのOSがLP64データモデルを採用した場合に`long`とポインタのサイズが64ビットになる。`long`が64ビットのつもりで使ったら32ビットで値が入りきらなかった、ということは十分起こり得るため、64ビット以上が必要なら`long long`を選択すること。

`long long`は64ビットである。128ビットCPUの時代は当面必要ないからこないのと、コンパイラ拡張の組み込み整数型として`__int128`型および`__uint128`型が定義される場合があるが標準の範囲では汎用CPUの多くが128ビット整数型をサポートしない限りそれを標準サポートはしないためである。

```cpp
// C++20
#include <iostream>
#include <limits>
#include <type_traits>
#include <string_view>

template <class T>
struct to_integer : std::type_identity<T> {};

template <>
struct to_integer<char> : std::type_identity<int> {};

template <class T>
void print_range(std::string_view name) {
    using IntegerT = typename to_integer<T>::type;

    std::cout << name;
    std::cout << " signed:["
              << static_cast<IntegerT>(std::numeric_limits<T>::min()) << ", "
              << static_cast<IntegerT>(std::numeric_limits<T>::max()) << ']';

    std::cout << " unsigned:[0, "
              << static_cast<std::make_unsigned_t<IntegerT>>(
                  std::numeric_limits<std::make_unsigned_t<T>>::max()
                 ) << ']'
              << std::endl;
}

int main() {
    print_range<char>("char");
    print_range<short>("short");
    print_range<int>("int");
    print_range<long long>("long long");
}
```


### 符号なし整数型のビット表現
符号なし整数型のビット表現はシンプルである。値が1足されるたびに一番下のビット (LSB) に1が加算され、上のビットは順に繰り上がっていく。

8ビット整数型の場合、各値のビット表現は以下のようになる：

| 値 | ビット表現 |
|----|------------|
| 1            | `0b0000'0001` |
| 2            | `0b0000'0010` |
| 4            | `0b0000'0100` |
| 127          | `0b0111'1111` |
| 128          | `0b1000'0000` |
| 255 (最大値) | `0b1111'1111` |

```cpp
#include <iostream>
#include <bitset>

void print_bit(unsigned int value) {
    std::cout << "0b" << std::bitset<8>{value}.to_string() << std::endl;
}

int main() {
    print_bit(1);
    print_bit(2);
    print_bit(4);
    print_bit(127);
    print_bit(128);
    print_bit(255);
}
```

値255のビット表現`0b1111'1111`に対してさらに1を加算すると、`0b1'0000'0000`となる。そして8ビットでは繰り上がった最上位ビットを表現できないため切り捨てられて結果として値0 (`0b0000'0000`) となる。これが符号なし整数型のオーバーフローである。


### 符号付き整数型のビット表現
符号付き整数型は、実装方法としてはいくつかの種類があるが、標準C++では「2の補数表現」であると規定される。

これは、負数を表現するために「絶対値をビット反転した値に+1する」という方式である。この方式の優れているところは、正数も負数も同じ方式で演算できる、減算を加算で表現できるなどがあるが、そのほかに「値`-0`を表現するビット列が存在しない」というものがある。

値0 (`0b0000'0000`) をビット反転`0b1111'1111`して+1すると`0b0000'0000`になる。そのために`-0`を表現しようとしても`+0`のビットになる。

これによって符号付き整数型が全順序をもつことが規定される。

たとえば浮動小数点数型は全順序ではない。NaNの存在も理由のひとつではあるが、`-0.0`と`+0.0`が存在するために`-0.0 == +0.0`が`true`かつ`-0.0 < +0.0`が`false`となる、つまり異なる値同士が同値であると見なされるために、型のすべての値に順序関係があると言えないためである。

特定の値を特別視する必要がないことは、演算速度の向上を期待できることはもちろん、ハッシュ値を求める際に一意性があると言えることなど、多くの面でよい効果がある。

2の補数表現であるという特徴上、「最大値に1を加算することで負数の最小値になる」のは自明な動作である。値127 (`0b0111'1111`) を+1すると、値-128 (`0b1000'0000`) になる。しかし、コンパイラとしてはこの動作は多くのプログラマが望むものではないとして積極的に最適化し、加算を続けても正数のままであるという認識のコードを生成する。そのため、C++の規格としては、符号付き整数型は2の補数表現であると規定されているにも関わらず、オーバーフローは未定義動作のままである。


## 整数型の別名
組み込み整数型として`int`は「多くの状況で十分な大きさの整数型」としてデフォルトで選択すべき整数型である。

しかし、用途に特化して選択すべき整数型がほかにたくさんある。

### ビット幅規定の整数型
`<cstdint>`ヘッダでは、ビット幅が規定された整数型の別名が定義される。

これらは、以下のような用途に適している：

- シリアライズ (言語間通信、異種環境間の通信など、異なる環境とやりとりするためにはビット幅が決まっていたほうが扱いやすい)
- 値の幅を想定できる (値が十分に小さく8ビットで十分なら`uint8_t`、値が大きく不特定多数が入力される場合は`uint64_t`など)

ただし`uint8_t`は`unsigned char`の別名であるため、`std::printf()`や`std::cout`などで出力すると数値ではなく文字が出力されてしまう。`int`などにキャストして使用すること。


### ポインタサイズの整数
`<cstdint>`には、`uintptr_t`と`intptr_t`という整数型が定義される。これらは、ポインタと同じサイズの整数型である。いくつかの状況で、ポインタに整数を代入したい、整数にポインタを代入したい、ということが発生する。

たとえばイベントシステムにおいて、`void*`型でひとつだけ任意の引数をイベントにもたせることができる、というような状況だ。

```cpp
void on_click_button(void* arg);
```

このイベントに単純なボタンのIDをもたせたい場合、ポインタに整数を代入させることが考えられる。ポインタはアドレス値をもつ型であるため整数型の一種であると考えられるためだ。そのようなときに、ポインタサイズの整数型があると便利である。ポインタがどのようなサイズをもっているかは環境によるためだ。

```cpp
// イベントを起動する側
std::uintptr_t button_id = 3;
void* arg = reinterpret_cast<void*>(button_id); // 整数をポインタに変換して引数として渡す
on_click_button(arg);
```

```cpp
// イベント側
void on_click_button(void* arg) {
    // 受け取った引数を整数に変換して使用する
    std::uintptr_t button_id = reinterpret_cast<std::uintptr_t>(arg);
}
```


### UNIX時間
日時は「2020年3月1日 15時30分20秒」のような形式だが、これは内部的にはある時間点からの経過時間として整数で表現できる。そのために`<ctime>`では`std::time_t`という整数型が定義される。

この型は199x年代までは32ビットとして実装されていたが、2038年にオーバーフローしてしまうことがわかり、現在は64ビットとして実装される。64ビットでは、秒単位であれば西暦6000億年くらいまでなら扱える。マイクロ秒単位でも30万年ほどは扱える。

日時を年、月、日などで分けて記録すると多数のフィールドが必要になるが、整数表現で記録すれば64ビット整数ひとつのフィールドで済む。


### 配列の要素数はどのような整数型であるべきか
標準C++において、配列の要素数は`<cstddef>`で定義される符号なし整数`std::size_t`として定義される。これはいくつかの不便を引き起こしている。

```cpp
for (int i = 0; i < c.size(); ++i)
```

このようなfor文を記述すると、`c.size()`によって返る整数型が符号なし、`i`が符号付きであるために、「符号付き整数と符号なし整数を比較しようとしました」という警告が発生する。

また、`max(0, c.size() - N)`のようにして要素数を減算した結果が0未満にならないようにするプログラムを書きたいことがあるが、要素数が符号なし整数であるために、0未満になるとオーバーフローしてしまって扱いにくいという問題がある。

そのため、さまざまな問題を経験した結果として「配列の要素数は符号付き整数であるほうが扱いやすい」と考えられる。標準C++には配列の要素数を符号付き整数として取得する`std::ssize()`関数が`<iterator>`で定義されるため、これを使用できる。

```cpp
for (int i = 0; i < std::ssize(c); ++i)
```

