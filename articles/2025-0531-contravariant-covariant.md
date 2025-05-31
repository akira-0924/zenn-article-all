---
title: "TypeScriptの共変、反変について本質を理解した気がするのでまとめる"
emoji: "✌️"
type: "tech"
topics: [TypeScript]
published: false
---

TypeScriptの**共変（Covariance）** と**反変（Contravariance）** について、ちゃんと理解できた気がしたので、自分なりにまとめてみます。


## 戻り値の共変

まずは戻り値から。

```typescript
const fn1: () => User = () => admin;
const fn2: () => Admin = () => user;
```

fn1 は「User型の戻り値を期待しているが、実際にはAdmin型（Userのサブタイプ）を返している」のでOK。

fn2 は「Admin型の戻り値を期待しているが、実際にはUser型（Adminのスーパタイプ）を返している」のでNG。

ポイント：<br>
AdminはUserを包含しているので、Userを期待する場所でAdminを渡すのは問題ない（サブタイプをスーパータイプとして扱える）。
逆に、Adminを期待している場所にUserを渡すと、Admin特有のプロパティ（permissions）にアクセスしようとしたときにundefinedになるかもしれないのでNG。


## 引数の反変

次は引数です。
```typescript
const fn3: (arg: User) => void = (arg: Admin) => console.log(arg.name, arg.permissions);
const fn4: (arg: Admin) => void = (arg: User) => console.log(arg.name);
```
見逆のように見えるかもしれませんが、以下のように考えると共変と同じように捉えることができます。

callFn と callFn2 の例
```typescript
const callFn = (fn: (arg: User) => void) => {
  const user: User = { name: "Bob" };
  fn(user);
};

const callFn2 = (fn: (arg: Admin) => void) => {
  const admin: Admin = { name: "Bob", permissions: [] };
  fn(admin);
};

const impl1: (arg: Admin) => void = (arg) => {
  console.log(arg.name, arg.permissions.length);
};

const impl2: (arg: User) => void = (arg) => {
  console.log(arg.name);
};

// NG: impl1はAdminを期待しているが、callFnではUserしか渡されない
callFn(impl1);

// OK
callFn(impl2);

// OK
callFn2(impl1);

// OK: impl2はUserを引数に取るが、Adminを渡しても問題ない（AdminはUserを包含している）
callFn2(impl2);
```

ポイント

callFn(impl1) がNGな理由
	•	callFnは引数として「User型のオブジェクト」を渡す。
	•	impl1は引数で「Admin型のオブジェクト」を期待している。
	•	AdminはUserを包含しているので型的に互換性があるように見えるが、
	•	実際にはimpl1内部でarg.permissionsのようにAdmin特有のプロパティを参照しているかもしれない。
	•	しかし、callFnはUserしか渡さないので、permissionsがundefinedになってしまう可能性がある。
	•	よって、型安全的にはNGとなる。

callFn2(impl2) がOKな理由
	•	callFn2はAdminを渡す。
	•	impl2はUserを期待している。
	•	AdminはUserを包含しているので、Userを期待する実装（impl2）にAdminを渡しても問題ない。
	•	Admin特有のプロパティは参照しないので型的に安全。



## まとめ
	•	戻り値は共変（スーパータイプ → サブタイプOK）
→ AdminはUserを包含しているので、Userを期待するところでAdminを返すのはOK。
	•	引数は反変（サブタイプ → スーパータイプOK）
→ Userを引数に取る関数に、Adminを渡すのはOK（AdminはUserのスーパーセットなのでUserのプロパティも持っている）。
→ 逆にAdminを引数に取る関数に、Userしか渡さないとAdmin固有のプロパティを使えない可能性があるのでNG。

実装側がどんなプロパティを期待しているか（実装側がどこまで責務を持っているか）が大事ですね。
