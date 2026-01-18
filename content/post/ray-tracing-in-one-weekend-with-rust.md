---
title: "Ray Tracing in One Weekend を Rust で実装してみた"
description: 
slug: ray-tracing-in-one-weekend-with-rust
date: 2026-01-18
image: img/ray-tracing-in-one-weekend-with-rust/ray_tracing_in_one_weekend.png
categories:
  - Rust
tags: [Rust, raytracing]
---

## はじめに

『プログラミング Rust』 をある程度読み進めたので、Rust の実装の練習がしたいと思い、ずっと積読していた [Ray Tracing in One Weekend](https://raytracing.github.io/books/RayTracingInOneWeekend.html) をやりました。
この本ではもともとレイトレーシングプログラムを C++ で実装していくのですが、それほど高度な言語機能は利用されていないため理解が比較的容易で、Rust に移植するにはおすすめの題材かと思います。

この本を最後まで進めることで、アイキャッチのような画像を生成することができます。
コードは以下のリポジトリに置いてあるので、もしよければご覧ください。

<div class="iframely-embed"><div class="iframely-responsive" style="height: 140px; padding-bottom: 0;"><a href="https://github.com/irimame/ray-tracing-in-one-weekend" data-iframely-url="https://iframely.net/DgT3UB17?card=small&theme=dark"></a></div></div><script async src="https://iframely.net/embed.js"></script>

ちなみにですがこの本には[邦訳版](https://inzkyk.xyz/ray_tracing_in_one_weekend/)もあるみたいです。内容は確認していませんが...

## 内容

この本の主要部分は13の章からなります。

1~3章で3次元ベクトル `Vec3`、点を表す `Point3`、RGB値を格納する `Color3` などの基礎的な構造体を定義します。私は `Vec3` を3つの `f64` を持つタプル型構造体として定義し、`Vec3` を持つニュータイプパターンとして `Point3` と `Color3` を定義しました。

```rust
struct Vec3(f64, f64, f64);
struct Point3(Vec3);
struct Color3(Vec3);
```

演算子オーバーロードは同じ型同士の2項演算や `f64` によるスカラー倍はもちろん、点から点を引くとベクトルになるという物理的解釈に基づき `Point3` 同士の減算は `Vec3` を返すように `Sub` トレイトを実装するなどしました。定義する演算がやや多く煩雑ではありましたが、演算子関係のトレイトの実装のよい練習になりました。
あとは `Vec3` に内積 $\bm{a} \cdot \bm{b}$、クロス積 $\bm{a} \times \bm{b}$、ノルムを計算する型関連関数 (C++ でいうところの static メンバ関数) を実装したりしました。ここはメソッドとして定義してもよいかもしれません。

4章で光線を表す `Ray` とカメラ (光線の発射点) を表す `Camera` を定義します。viewport 上にピクセルを配置し、カメラからピクセルの方向に光線を発射します。その途中に物体があると光線が反射、散乱、屈折して軌道が変化します。また光は散乱するごとに減衰します。それらを計算していった結果が最終的にレンダリング画像のピクセルとして記録されます。では光線の軌道変化や減衰はどうやって計算するのか？という疑問が湧いてきますが、これは以降の章で説明されています。

5~8章は光線が衝突する物体 `Hittable` の1つの形状として球 `Sphere` を定義し、球に当たった光線が描く軌跡を計算していきます。私は `Hittable` をトレイトとして定義し、`Sphere` を `Hittable` を実装した構造体として定義しました。
さて、光線が球に当たるかを判定するには、球の方程式と光線の式が交点を持つかどうかを判定する必要があります。これは2次方程式が解を持つかどうかを判定すればよく、したがって判別式を計算すればよいことになります。また、球面上の光線が当たった点とその点における法線ベクトルを計算することで、以降の章で光線が次にどの方向に飛んでいくか、そもそも光線は球の外側と内側どちらから当たったのか (屈折の計算で必要になります) を計算できるようにします。これらの情報は `HitRecord` 構造体に格納します。

```rust
// in hittable.rs
#[derive(Clone)]
pub struct HitRecord {
    pub p: Point3,
    pub normal: Vec3,
    pub mat: Rc<dyn Material>,
    pub t: f64,
    pub front_face: bool,
}

pub trait Hittable {
    fn hit(&self, ray: &Ray, ray_t: Interval) -> Option<HitRecord>;
}
```

9~11章では物体の材質 `Material` として Lambert 反射体 `Lambertian`、金属 `Metal`、誘電体 `Dielectric` を定義し、材質ごとに異なる光の散乱の計算方法を実装していきます。ここでは `Material` をトレイトとして実装し、`Lambertian`, `Metal`, `Dielectric` を `Material` を実装した構造体として定義しました。Lambert 反射体は非一様な散乱分布をもち、金属 (鏡など) は反射の法則に従い、誘電体 (ガラスなど) は Snell の法則に従って光を屈折させます。個人的に印象に残ったのは、Lambert 反射体の散乱方向の計算でした。Lambert 反射体は入射光の方向と法線ベクトルのなす角を $\theta$ とするとき、 $\cos{\theta}$ に比例した散乱分布を持ちます。一様分布であれば適当に乱数を生成するだけでよいのですが、今回は非一様分布なのでいくつかの計算ステップが必要なのかと思っていたら、実はランダム方向の単位ベクトルに単位法線ベクトルを足し合わせるだけで分布を再現できるのです。このアイデアはシンプルかつ鮮やかで、感動すら覚えました。

12,13章では話題がカメラに移り、カメラの位置、姿勢、視野角を自由に設定できるようにしたり、Defocus blur というテクニックによって被写界深度を表現できるようにします。
カメラ座標系を導入することで、カメラの位置や姿勢を変えてもピクセルの位置を容易に計算できるようにします。ちなみに `Vec3` の型関連関数として導入したクロス積はカメラ座標系を計算するときに活躍します。Defocus blur は私が本の説明をまともに理解できていないのでアレですが、光線の発射位置にレンズの焦点距離に応じたばらつきを加えることで、被写界深度を再現する方法なのかもしれないと思っています (カメラについては何もわからないので適当なことを言っていたらすみません)。

最後にアイキャッチのような多数の球の画像を生成するのですが、手元の環境ではリリースビルドでも10分強かかりました。ただしこれはシングルスレッドでの実行なので、並列化したらどうなるかは気になります。そのあたりの知識がついたらやってみたいと思います。

## 実装上の反省

関数のシグネチャに必要以上に参照を付けすぎたせいでコードの記述がやや煩雑になってしまった感があります。
例えば `Vec3` は 3つの `f64` の値からなる型なのでコピーのコストはごく僅かと考えてよく、実際 `Copy` を実装していました。しかし、以下に代表される `Vec3` の型関連関数では参照渡しを多用していました。この影響で実引数に `&` を付けなければならなかったのが面倒でした。これは値渡しで十分でしたね...。

```rust
// in vec3.rs
pub fn unit_vector(vec: &Vec3) -> Vec3 {
    *vec / vec.length()
}

pub fn cross(u: &Vec3, v: &Vec3) -> Vec3 {
    Vec3(
        u.1 * v.2 - u.2 * v.1,
        u.2 * v.0 - u.0 * v.2,
        u.0 * v.1 - u.1 * v.0,
    )
}

// in camera.rs
pub struct Camera {
    //...
    pub lookfrom: Point3,
    pub lookat: Point3,
    pub vup: Vec3,
    u: Vec3,
    v: Vec3,
    w: Vec3,
    //...
}

impl Camera {
    //...
    // Calculate the u,v,w unit basis vectors for the camera coordinate frame.
    self.w = Vec3::unit_vector(&(self.lookfrom - self.lookat));
    self.u = Vec3::unit_vector(&Vec3::cross(&self.vup, &self.w));
    self.v = Vec3::unit_vector(&Vec3::cross(&self.w, &self.u));
    //...
}
```

また、本の C++ 実装では、`HitRecord` 型の未初期化変数の参照を `Hittable::hit()` 関数の引数に渡し、関数側で値を設定するという典型的な処理が行われていましたが、これは Rust では禁止されています (コンパイラにも怒られた)。同じようなことをやりたければ、関数から `Option<T>` の形で返してもらうのがよさそうです。`Option<T>` は `T` を完全な形で初期化することができない場合にとりあえず `None` を入れておくことができる点でも便利です。

## おわりに

Ray Tracing in One Weekend の内容を Rust で実装しました。
この本には[続編](https://raytracing.github.io/books/RayTracingTheNextWeek.html)があり、これまでに作ったプログラムをさらに発展させていく形で話が進んでいくようです。レイトレーシングをもっと深く知りたいので、引き続き取り組んでいこうと思います。
