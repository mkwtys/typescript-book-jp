# Classes Emit

### IIFE\(Immediately-Invoked Function Expression\): 即時実行関数式

クラスの機能を実現するためにコンパイル後に生成されるJavaScriptは、以下のようなものかもしれません:

```typescript
function Point(x, y) {
    this.x = x;
    this.y = y;
}
Point.prototype.add = function (point) {
    return new Point(this.x + point.x, this.y + point.y);
};
```

TypeScriptが生成するクラスは、Immediately-Invoked Function Expression\(IIFE\)に包まれています。IIFEの例:

```typescript
(function () {

    // BODY

    return Point;
})();
```

クラスがIIFEに包まれている理由は、継承に関係しています。TypeScriptは、IIFEによってベースクラスを変数`_super`として補足（キャプチャ）できるようにしています:

```typescript
var Point3D = (function (_super) {
    __extends(Point3D, _super);
    function Point3D(x, y, z) {
        _super.call(this, x, y);
        this.z = z;
    }
    Point3D.prototype.add = function (point) {
        var point2D = _super.prototype.add.call(this, point);
        return new Point3D(point2D.x, point2D.y, this.z + point.z);
    };
    return Point3D;
})(Point);
```

TypeScriptは、IIFEによって、簡単にベースクラス `Point`を`_super`変数に捕捉できるようにしており、かつ、それがクラスの本体で一貫的に使用されていることに注意してください。

## `__extends`

クラスを継承するとTypeScriptが次のような関数を生成することに気づくでしょう:

```typescript
var __extends = this.__extends || function (d, b) {
    for (var p in b) if (b.hasOwnProperty(p)) d[p] = b[p];
    function __() { this.constructor = d; }
    __.prototype = b.prototype;
    d.prototype = new __();
};
```

ここで `d`は派生クラス\(**d**erived class\)を参照し、`b`はベースクラス\(**b**ase class\)を参照します。この関数は次の2つのことを行います：

1. 親クラスの静的メンバを子クラスにコピーする:`for (var p in b) if (b.hasOwnProperty(p)) d[p] = b[p];`
2. 子クラス関数のプロトタイプを準備し、必要に応じて親の`proto`のメンバを検索できるようにする。つまり、実質的に `d.prototype.__proto__ = b.prototype` です。

1を理解するのに苦労する人はほとんどいませんが、2については多くの人が理解に苦しみます。そのため、順番に説明します。

### `d.prototype.__proto__ = b.prototype`

これについて多くの人を教えた結果、次のような説明が最もシンプルだと分かりました。まず、`__extends`のコードが、単純な`d.prototype.__proto__ = b.prototype`とどうして同じなのか、そしてなぜ、この行それ自体が重要であるのかを説明します。これをすべて理解するためには、これらのことを理解する必要があります:

1. `__proto__`
2. `prototype`
3. `new`の関数の内側の`this`に対する効果
4. `new`の`prototype`と`__proto__`に対する効果

JavaScriptのすべてのオブジェクトは `__proto__`メンバを含んでいます。このメンバは古いブラウザではアクセスできないことがよくあります\(ドキュメントでは、この魔法のプロパティを `[[prototype]]`と呼ぶことがあります\)。これは1つの目的を持っています：検索しているプロパティがオブジェクトに見つからない場合\(例えば `obj.property`\)、`obj.__proto__.property`を検索します。それでもまだ見つからなければ、 `obj.__proto__.__proto__.property`を検索します： それが見つかるか、最後の`.__proto__`自体が`null`となるまで続きます。これは、JavaScriptが_プロトタイプ継承_\(prototypal inheritance\)をサポートしているということです。次の例でこれを示します。chromeコンソールまたはNode.jsで実行できます:

```typescript
var foo = {}

// foo を設定します foo.__proto__ も同様です
foo.bar = 123;
foo.__proto__.bar = 456;

console.log(foo.bar); // 123
delete foo.bar; // オブジェクトから削除します
console.log(foo.bar); // 456
delete foo.__proto__.bar; // foo.__proto__ から削除します
console.log(foo.bar); // undefined
```

これで、あなたは `__proto__`を理解できました。もう1つの便利な事実は、JavaScriptの関数\(`function`\)には`prototype`というプロパティがあり、そして、その`constructor`メンバは、逆に関数を参照しているということです。これを以下に示します:

```typescript
function Foo() { }
console.log(Foo.prototype); // {} 例: これは存在しており、undefinedではありません
console.log(Foo.prototype.constructor === Foo); // `constructor` というメンバがあり、逆に関数を指しています
```

次に、呼び出された関数内の`this`に対して`new`が及ぼす影響を見てみましょう。基本的に、`new`を使って呼び出された関数の内側の`this`は、関数から返される新しく生成されたオブジェクトを参照します。関数内で`this`のプロパティを変更するのは簡単です：

```typescript
function Foo() {
    this.bar = 123;
}

// new 演算子を使います
var newFoo = new Foo();
console.log(newFoo.bar); // 123
```

ここで理解しておくべきことは、関数に対する`new`の呼び出しにより、関数の`prototype`が、新しく生成されたオブジェクトの`__proto__`に設定されることです。次のコードを実行することによって、それを完全に理解できます:

```typescript
function Foo() { }

var foo = new Foo();

console.log(foo.__proto__ === Foo.prototype); // True!
```

これだけのことです。今度は`__extends`の抜粋を見てください。これらの行に番号を振りました：

```typescript
1  function __() { this.constructor = d; }
2   __.prototype = b.prototype;
3   d.prototype = new __();
```

この関数を逆から見ると、3行目の`d.prototype = new __()`は、 `d.prototype = {__proto__ : __.prototype}`を意味します\(`prototype`と`__proto__`に対する`new`の効果によるものです\)。それを2行目\(`__.prototype = b.prototype;`\)と組み合わせると、`d.prototype = {__proto__ : b.prototype}`となります。

しかし、待ってください。私達は、単に`d.prototype.__proto__`だけが変更され、`d.prototype.constructor`は、それまで通り維持されることを望んでいました。そこで重要な意味があるのが、最初の行\(`function __() { this.constructor = d; }`\)です。これによって`d.prototype = {__proto__ : __.prototype, constructor : d}`と同じことを実現できます\(これは関数の内側の`this`に対する`new`による効果のためです\)。したがって元の`d.prototype.constructor`を復元しているので、ここで変更されたものは、`__proto__`だけです。なので`d.prototype.__proto__ = b.prototype`となります。

### `d.prototype.__proto__ = b.prototype`の意味

これを行うことによって、子クラスにメンバ関数を追加しつつ、その他のメンバは基本クラスから継承することができます。次の簡単な例で説明します:

```typescript
function Animal() { }
Animal.prototype.walk = function () { console.log('walk') };

function Bird() { }
Bird.prototype.__proto__ = Animal.prototype;
Bird.prototype.fly = function () { console.log('fly') };

var bird = new Bird();
bird.walk();
bird.fly();
```

基本的に`bird.fly`は`bird.__proto__.fly`\(`new`は`bird.__proto__`の参照を`Bird.prototype`に設定することを思い出してください\)から検索され、`bird.walk`\(継承されたメンバ\)は`bird.__proto__.__proto__.walk`から検索されます\(`bird.__proto__ == Bird.prototype`、そして、`bird.__proto__.__proto__` == `Animal.prototype`です\)。

