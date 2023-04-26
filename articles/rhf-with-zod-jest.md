---
title: "React Hook FormとZodを使ったフォームのバリデーションとテスト"
emoji: "🍕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["reacthookform", "zod", "react", "test", "storybook"]
published: false
---

# はじめに

React Hook Form(RHF)は、React のフォームライブラリです。
基本的には非制御コンポーネント前提で、RHF 街提供している API を通して操作を行います。

今回は、RHF を使っていく上で細かいバリデーションやエラー表示などについてはまったポイントを中心に、Zod とテストを組み合わせてまとめていきたいと思います。

# 基本的な使い方

RHF を使った基本的なパスワード入力フォームを作成してみます。
フォームとしては現在のパスワード、新しいパスワード、新しいパスワード(確認用)の 3 つを入力するフォームです。
バリデーションの発火タイミングとしては、onBlur に指定します。
これにより、各要素のフォーカスが失ったタイミングでバリデーションが発火されます。
https://react-hook-form.com/api/useform/#mode

```tsx: password.tsx
import { useForm } from "react-hook-form";

const PasswordForm = () => {
  const { register, handleSubmit, formState: {errors} } = useForm({
    mode: 'onBlur',
  });

  const onSubmit = (data) => {
    console.log(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input type="password" {...register("currentPassword")} />
      <input type="password" {...register("newPassword")} />
      {errors.newPassword && <p>{errors.newPassword.message as string}</p>}
      <input type="password" {...register("confirmPassword")} />
      {errors.confirmPassword && <p>{errors.confirmPassword.message as string}</p>}
      <button type="submit">Submit</button>
    </form>
  )
}
```
