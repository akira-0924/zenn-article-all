---
title: "useFormでバリデーション制御に悩んだ話 〜setErrorとisValid〜"
emoji: "😀"
type: "tech"
topics: [React,ReactHookForm]
published: false
---

## TL;DR
- setErrorするだけではisValidが変わらない
- 素直にregisterやControllerのonChangeを使おう

## 目次

### はじめに
React Hook Form（以下 RHF）は、軽量で直感的に使えるフォームライブラリとして、多くのReactプロジェクトで利用されています。実際に使ってみると、register や Controller を通して簡単にバリデーションを設定でき、フォームの状態も formState を通じて一括管理できるので非常に便利です。

しかし、実際の開発では「ちょっとした例外的なケース」にぶつかることがあります。今回私が遭遇したのはその一つ。
setError を使って手動でエラーを出したのに、isValid がなぜか true のままになってしまう──という現象です。

バリデーションエラーが表示されているのに、送信ボタンが押せてしまう…
フォームの見た目と状態が一致しないもどかしさに、思わずコードとにらめっこしてしまいました。

この記事では、この現象を再現したシンプルなコードと、調査・検証の過程を共有します。React Hook Form を使っていて、「あれ？なんでこれ通っちゃうの？」と感じた方のヒントになれば嬉しいです。

### 今回のケース
改めて、今回遭遇してケースをシンプルにした形で整理します。
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
3. エラー表示は無くなるが、`clearError`はフォームに対してsubscribrしているわけではないので、フォーム全体の状態を監視して判定する`isValid`は切り替わらない
4. ボタンは非活性のままになる

ということになります。

### 解決策
- shouldValidate
- trigger()を挟んで再評価する
- registerを使う
- Controllerのrulesでバリデーションを実行