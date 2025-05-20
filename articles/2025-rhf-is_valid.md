---
title: "useFormでバリデーション制御に悩んだ話 〜setErrorとisValid〜"
emoji: "😀"
type: "tech"
topics: [React,ReactHookForm]
published: true
publication_name: atamaplus
---

## TL;DR
- setErrorするだけではisValidが変わらない
- 素直にregisterやControllerのonChangeを使おう

## はじめに
React Hook Form（以下 RHF）は、軽量で直感的に使えるフォームライブラリとして、多くのReactプロジェクトで利用されています。実際に使ってみると、register や Controller を通して簡単にバリデーションを設定でき、フォームの状態も formState を通じて一括管理できるので非常に便利です。

しかし、実際の開発では「ちょっとした例外的なケース」にぶつかることがあります。今回私が遭遇したのはその一つ。
setError を使って手動でエラーを出したのに、isValid がなぜか true のままになってしまう、という現象です。

バリデーションエラーが表示されているのに、送信ボタンが押せてしまう…
フォームの見た目と状態が一致しないもどかしさに、思わずコードとにらめっこしてしまいました。

この記事では、この現象を再現したシンプルなコードと、調査・検証の過程を共有します。React Hook Form を使っていて、「あれ？なんでこれ通っちゃうの？」と感じた方のヒントになれば嬉しいです。

## 今回のケース
改めて、今回遭遇したケースをシンプルにした形で整理します。
- RHFを使ってバリデーション付きのフォーム機能を実装
- Inputが1つあり、文字数の制限、必須項目としてバリデーションを設定し、エラーを表示したい
- 送信ボタンはエラーがある場合は押せないようにしたい

ざっくりこんな感じです。よくあるパターンかと思います。
そして自分が最初に書いていたコードが以下になります。(Chakra UIを使用しています)
```tsx
import React from 'react'
import { Box, Button, Input } from '@chakra-ui/react'
import { useForm } from 'react-hook-form'

type FormData = {
  name: string
}

export const Form4 = () => {
  const {
    setValue,
    getValues,
    handleSubmit,
    setError,
    clearErrors,
    formState: { errors, isValid }
  } = useForm<FormData>({
    mode: 'all',
    defaultValues: { name: '' }
  })

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value

    if (value.length > 5) {
      setError('name', { message: '5文字以内で入力してください。' })
    } else if (value.length === 0) {
      setError('name', { message: '必須項目です。' })
    } else {
      clearErrors('name')
    }
    setValue('name', value)
  }

  const onSubmit = (data: FormData) => {}

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <Input
        w="50%"
        name="name"
        value={getValues('name')}
        placeholder="5文字以内"
        onChange={handleChange}
      />
      <Box mt={2}>
        {errors.name && <p style={{ color: 'red' }}>{errors.name.message}</p>}
      </Box>
      <Button type="submit" bg="blue.500" color="white" disabled={!isValid}>
        Submit
      </Button>
    </form>
  )
}
```
- useFormをhookを使って`setValue`や`getValues`、`setError`や`clearErrors`をしていて、状態を管理しています。
- `formState`プロパティから`isValid`を取得して、送信ボタンの`disabled`属性を制御している。
- `formState`プロパティから`errors`を取得して、エラーがある場合はエラーメッセージを表示している。

バリデーションロジックや複数項目など実際はもう少し複雑かと思いますが、シンプルにすると大体こんな感じかなと思います。
コードだけ見ると一見正常に動きそうですが、実際には**一度エラーになるとisValidがfalseのまま切り変わらず送信ボタンが押せなくなってしまいます。**

![a](/images/rhf-1.gif)

なぜ...？と思い、公式ドキュメントを見ていたところ原因がわかりました。
`isValid`はフォーム全体の状態を判定してtrue/falseを返します。
しかし、`clearErrors`ではisValidには影響を与えません。

https://react-hook-form.com/docs/useform/clearerrors

> This will not affect the validation rules attached to each inputs.

そして、`setError`では強制的に`isValid`をfalseにします。

https://react-hook-form.com/docs/useform/seterror

> This method will force set isValid formState to false. However, it's important to be aware that isValid will always be derived from the validation result of your input registration rules or schema result.

つまり、この場合だと、
1. カスタムのバリデーションに引っかかった場合は`setError`して`isValid`を強制的に`false`に切り替える
2. InputのonChangeでバリデーションに引っかからない正常な値になった時に`clearError`して、errorsオブジェクトを空にする。
3. エラー表示は無くなるが、`clearError`はフォームに対してsubscribeしているわけではないので、フォーム全体の状態を監視して判定する`isValid`は切り替わらない
4. ボタンは非活性のままになる

ということになります。

## どうやって解決するか
ではどうすればよかったのか。色々試しました。結論としては、`register()`を使ってフォーム値として登録すれば良いのですが、他にも方法があるのではと思い検証してみました。

### 1. setValueするときに`shouldValidate`オプションを使ってみる

まず試してみたことがこれです。

https://react-hook-form.com/docs/useform/setvalue

>Whether to compute if your input is valid or not (subscribed to errors).
Whether to compute if your entire form is valid or not (subscribed to isValid).

一度エラーになった後`isValid`がfalseのままなのが問題なら、その後正常な値をsetValueする時に`shouldValidate: true`オプションを使えば再評価されて切り替わるんじゃないか？と思いました。

```tsx
if (value.length > 5) {
      setError('name', { message: '5文字以内で入力してください。' })
    } else if (value.length === 0) {
      setError('name', { message: '必須項目です。' })
    } else {
      clearErrors('name')
    }
    setValue('name', value, { shouldValidate: true })
    // setValue('name', value)
```

しかし、実際の挙動は以下でした。

![b](/images/rhf-2.gif)

はい。ボタンがずっと活性状態です。コンソールを見るとわかるのですが、`setError`では確かにisValidがfalseになっています。が、その後すぐにtrueに切り替わっています。そしてerrorsオブジェクトは期待通り更新されたままです。<br>
これも結局、setErrorしただけでは、フォームに対して値を登録しているわけではないので、isValidに影響がないという仕様が関係しています。ドキュメントよるとsubscribed to isValidと書いてあり、`shouldValidate: true`で確かにフォーム対して登録しているのですが、現状のコードではバリエーション自体はuseFormの管理下にありません。onChangeイベントのif-else文で独自で書いた分岐処理をバリデーションとしているだけであって、それがuseFormに登録されているわけではないのです。なので、onChangeで最後に`setValue`をしてもuseForm側からすれば、"どんな値でもOK"という判定になり、isValidは必ずtrueに切り替わるということです。これではダメですね...

### 2. `trigger()`を挟んで再評価する

https://react-hook-form.com/docs/useform/trigger

ここまで読んでいただいた方はなんとなく察しがつくかと思いますが、これも1の`shouldValidate: true`と同様の動きになります。なぜならカスタムのバリデーションがuseFormの管理下にあらず、内部的にはバリデーションなしのどんな値でもOKな状態なので、`trigger()`でフォームの値を再評価しても`isValid`は必ずtrueに切り替わるだけです。なのでこれもダメでした...

### 3. registerを使う

https://react-hook-form.com/docs/useform/register

次は`register()`使ってフォームに値を登録する方法です。
register を使うことで、React Hook Form に対して input 要素とバリデーションルールの紐付けが明示的に行われます。そのため、値が変化した際に自動でバリデーションが実行され、isValid が正しく反映されます。
registerメソッドには第二引数でオプションが指定できます。下記のようにバリデーションを登録しました。

```tsx
<Input
  w="50%"
  placeholder="5文字以内"
  type="text"
  {...register('name', {
    required: '必須項目です',
    maxLength: {
      value: 5,
      message: '5文字以内で入力してください'
    }
  })}
/>
```

![c](/images/rhf-3.gif)

これはうまく動いてそうです。registerを使ったからというより、バリデーションをuseFormの監視下に登録したということですね。

### 4. Controllerのrulesでバリデーションを実行

最後にRHFの`Controller`コンポーネントを使う方法です。

https://react-hook-form.com/docs/usecontroller/controller


```tsx
<Controller
  control={control}
  name="name"
  rules={{
    maxLength: { value: 5, message: '5文字以内で入力してください' },
    required: { value: true, message: '必須項目です' }
  }}
  render={({ field: { value, onChange }, fieldState: { error } }) => (
    <>
      <Input
        value={value ?? ''}
        w="50%"
        placeholder="5文字以内"
        type="text"
        onChange={onChange}
      />
      <Box mt={2}>
        {error && <p style={{ color: 'red' }}>{error.message}</p>}
      </Box>
    </>
  )}
/>
```

これは、`<Controller></Controller>`で囲い、useFormから`control`プロパティを受け取りそれをそのままControllerのcontrol propsに渡してあげます。（これで監視下にするみたいなイメージです）
<br>そして、`render()`の中で該当のInputをレンダリングします。Controllerには`rules`propsが用意されており、ここにバリデーションを設定することができます。

この場合は、render()の引数から`field.value`や`fieldState.error`を取得できるので、そのまま値をセットできます。onChangeも独自で関数を定義して、registerやsetValueできますが、これも`field.onChange`を取得できるのでそのままInputのonChangeイベントに渡してあげると楽に実装できます。

ただし、一点だけ注意する部分あり、`field.onChange`を使う場合はuseFormの`mode`を`onChange` or `onBlur` or `all`等に変更してください。デフォルトが`onSubmit`になっており、送信ボタンを押さないとerrorsオブジェクトに値が入ってこないので調整が必要です。
```tsx
const {
  handleSubmit,
  getValues,
  control,
  formState: { isValid }
} = useForm<FormData>({
  mode: 'onChange',
  defaultValues: {
    name: 'テスト',
  }
})
```

## じゃあsetErrorっていつ使うの？

ここまでで、isValidとsetError、clearErrorについてみてきましたが、「isValidに影響を与えないのであればclearErrorやsetErrorってなんのためにあるの？」と疑問に思いました。
そこで、もう少しドキュメントを読み進めていると、

>Can be useful in the handleSubmit method when you want to give error feedback to a user after async validation. (ex: API returns validation errors)

とありました。

1つの使用例としてクライアント側のバリデーションではなくsubmitをしてAPIからエラーが返ってきた時にsetErrorしてerrorを表示したりできるよってことですね。コードで書くと以下のような感じでしょうか。
```tsx
const fakeApiResponse = {
  success: false,
  errors: {
    name: 'このnameは既に存在します'
  }
}

const onSubmit = (data: FormData) => {
  console.log('送信データ:', data)

  console.log(data);
  if (!fakeApiResponse.success) {
    Object.entries(fakeApiResponse.errors).forEach(([field, message]) => {
      setError(field as keyof FormData, {
        type: 'server',
        message
      })
    })
    return
  }

  alert('送信成功')
}
<form onSubmit={handleSubmit(onSubmit)}>
  <Input
    w="50%"
    placeholder="5文字以内"
    type="text"
    {...register('name', {
      required: '必須項目です',
      maxLength: {
        value: 5,
        message: '5文字以内で入力してください'
      }
    })}
  />
  <Box mt={2} mb={2}>
    {errors.name && <p style={{ color: 'red' }}>{errors.name.message}</p>}
  </Box>

  <Button type="submit" bg="blue.500" color="white" disabled={!isValid}>
    Submit
  </Button>
</form>
```

![d](/images/rhf-4.gif)

確かにuseFormでerrorの状態を管理する場合、APIレベルのエラーもuseFormのerrorsオブジェクトに追加しないと2重管理になってしまいますね...。だから`setError`してuseFormのerrorに直接登録するということですね!

（ちなみに入力するとバリデーションが再評価されるのでclearErrorは必要ありません。）

## まとめ
今回はuseFormのisValidとsetErrorの関係についてみてきました。
普通はregisterを使うことがほぼだと思うので、あまり気にすることはないと思うのですが、サーバー側のエラーをsetErrorするようにしたり、`shouldValidate`オプションだったり、知らなかったことも多かったので良い勉強になりました。
useFormのバリデーション周りで困っている方のヒントになれば幸いです。