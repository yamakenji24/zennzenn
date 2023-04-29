---
title: "React Hook FormとZodを使ったフォームのバリデーションとテスト"
emoji: "🍕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["reacthookform", "zod", "react", "test", "storybook"]
published: true
---

# はじめに

React Hook Form(RHF)は、React のフォームライブラリです。
基本的には非制御コンポーネント前提で、RHF 街提供している API を通して操作を行います。

今回は、RHF を使っていく上で細かいバリデーションやエラー表示などについてはまったポイントを中心に、Zod とテストを組み合わせてまとめていきたいと思います。

# 基本的な使い方

RHF を使った基本的なパスワード入力フォームを作成してみます。
フォームとしては現在のパスワード、新しいパスワード、新しいパスワード(確認用)の 3 つを入力するフォームです。
バリデーションの発火タイミングとしては、onBlur に指定します。
https://react-hook-form.com/api/useform/#mode

Zodで定義したSchemaを利用して、RHFのバリデーションに利用します。
これにより、各inputのフォーカスが外れた時にバリデーションによるエラーがレンダリングされます。

```tsx: Password.tsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import * as z from "zod";

const createPasswordSchema = () => {
  const password = z
    .object({
      currentPassword: z.string().min(1),
      newPassword: z.string().min(8),
      confirmPassword: z.string().min(1),
    })
    .superRefine(({currentPassword, newPassword, confirmPassword}, ctx) => {
      if (currentPassword === newPassword) {
        ctx.addIssue({
          path: ['newPassword'],
          code: 'custom',
          message: 'password is same with current password',
        });
      }
      if (newPassword !== confirmPassword) {
        ctx.addIssue({
          path: ['confirmPassword'],
          code: 'custom',
          message: 'password is not same',
        });
      }
    });

  return password;
};
type Password = z.infer<ReturnType<typeof createPasswordSchema>>;

export const PasswordForm = () => {
  const { register, handleSubmit, formState: {errors} } = useForm<Password>({
    resolver: zodResolver(createPasswordSchema())
    mode: 'onBlur',
  });

  const onSubmit = handleSubmit(({currentPassword, newPassword}) =>
    console.log("submti: ", currentPassword)
  );

  return (
    <form onSubmit={onSubmit}>
      <label htmlFor="currentPassword">Current Password</label>
      <input type="password" {...register("currentPassword")} />

      <label htmlFor="newPassword">New Password</label>
      <input type="password" {...register("newPassword")} />
      {errors.newPassword && <p>{errors.newPassword.message as string}</p>}

      <label htmlFor="confirmPassword">Confirm Password</label>
      <input type="password" {...register("confirmPassword")} />
      {errors.confirmPassword && <p>{errors.confirmPassword.message as string}</p>}
      <button type="submit">Submit</button>
    </form>
  )
}
```

```tsx: Password.stories.tsx
import type { ComponentStory, ComponentMeta } from '@storybook/react';
import { userEvent, within } from '@storybook/testing-library';
import { PasswordForm } from './password'

export default {
  title: 'PasswordForm',
  component: PasswordForm,
} as ComponentMeta<typeof PasswordForm>;

export const ValidPassword = {
  play: async({
    canvasElement
  }: { canvasElement: HTMLElement }): Promise<void> => {
    const canvas = within(canvasElement);

    userEvent.type(canvas.getByLabelText('Current Password'), 'password')
    userEvent.type(canvas.getByLabelText('New Password'), 'newpassword1')
    userEvent.type(canvas.getByLabelText('Confirm Password'), 'newpassword1')
    userEvent.tab();
  }
}
export const NewPasswordNotSame = {
  play: async(ctx): Promise<void> => {
    await ValidPassword.play?.(ctx)

    const canvas = within(ctx.canvasElement);
    userEvent.type(
      canvas.getByLabelText('Confirm Password'),
      '+1',
    );
    userEvent.tab();
  }
}
```

```tsx: Password.spec.tsx
import { composeStories } from '@storybook/testing-react';
import { render, renderHook, screen, waitFor } from '@testing-library/react';
import * as stories from './password.stories'

describe('password', () => {
  const { NewPasswordNotSame } = composeStories(stories);
  
  test('新しいパスワードとパスワード(確認)が一致しない時には、バリデーションエラーが表示される', async () => {
    const { container } = render(<NewPasswordNotSame />);
    await NewPasswordNotSame.play({canvasElement: container});
    await waitFor(() => expect(screen.getByText('password is not same')).toBeInTheDocument())
  })
})
```

# はまったところ
ここでの期待は、以下になります。
- 各要素に対してフォーカスが失った時にバリデーションのエラーは表示されて欲しい
- バリデーションエラーの表示後、正しい入力値を入力した後にはエラーは消えて欲しい

細かく動きを見た時に、以下の時に期待した動作をしていなさそうなのが判明しました。
1. 現在のパスワード、新しいパスワード、新しいパスワード(確認)を入力後、新しいパスワードを別の入力値にした時にパスワード(確認)にエラーが表示されない
2. 1の状態で、新しいパスワードの方をパスワード(確認)に合わせた入力値にした時に表示されていたエラーが消えない

基本、新しいパスワードを触った時には新しいパスワード(確認)も触ることが意図したもののため、エラーを出さなくてもユーザーの操作の流れ的に分かるものの、フォームの状態としてあるべき状態をそのままユーザーに伝えるのは大事です。

はまった点として、ZodのSchema的に、superRefine内で各要素に対してのバリデーションを行っているのでエラーも表示されそうですがされない点です。
RHFのドキュメントを見ていると、以下のような記述があるのを見つけました。
https://react-hook-form.com/api/useform/#resolver
```
Re-validation of an input will only occur one field at time during a user’s interaction. 
The lib itself will evaluate the error object to trigger a re-render accordingly.
```
つまり、バリデーションによるエラーの再レンダリングは一度に一つの要素にしか行われず、基本的にフォーカスしている要素に対してのみ行われるということ。
そのため、新しいパスワードの入力後にパスワード(確認)に対してのエラーの再レンダリングが行われなかったということです。

# 対応
useFormのregisterが持っているmethodのonBlurを利用します。
inputのfocusが外れた時に、別要素の入力値が空でない場合にその要素のバリデーションを発火させます。
これにより、例えば依存する要素に対して任意のタイミングでバリデーションを発火させ、再レンダリングを行えるようになります。

```tsx: password.tsx
  const { 
    ...,
    getValues, trigger
  } = useForm(...)
  return (
    <form onSubmit={onSubmit}>
      <label htmlFor="currentPassword">Current Password</label>
      <input type="password" {...register("currentPassword")} />

      <label htmlFor="newPassword">New Password</label>
      <input type="password" {...register("newPassword", {
        onBlur: () => {
          if (getValues('confirmPassword').length !== 0) {
            trigger('confirmPassword')
          }
        }
      })} />
      {errors.newPassword && <p>{errors.newPassword.message as string}</p>}

      <label htmlFor="confirmPassword">Confirm Password</label>
      <input type="password" {...register("confirmPassword")} />
      {errors.confirmPassword && <p>{errors.confirmPassword.message as string}</p>}
      <button type="submit">Submit</button>
    </form>
  )
```
Storybook上で確認、テストすると次のようになると思います。

```tsx: password.stories.tsx
export const RevalidationAfterInput = {
  play: async(ctx): Promise<void> => {
    await ValidPassword.play?.(ctx)

    const canvas = within(ctx.canvasElement);
    userEvent.type(
      canvas.getByLabelText('New Password'),
      '+1',
    );
    userEvent.tab();
  }
}

export const ErrorShouldDisapperWhenValidInput = {
   play: async(ctx): Promise<void> => {
    await RevalidationAfterInput.play?.(ctx)

    const canvas = within(ctx.canvasElement);
    userEvent.clear(canvas.getByLabelText('New Password'))
    userEvent.type(
      canvas.getByLabelText('New Password'),
      '+1',
    );
    userEvent.tab();
  }
}

```

```tsx: password.spec.tsx
describe('password', () => {
  const { RevalidationAfterInput, ErrorShouldDisapperWhenValidInput } = composeStories(stories);
  
  test('新しいパスワードとパスワード(確認)が一致しない時には、バリデーションエラーが表示される', async () => {
    const { container } = render(<RevalidationAfterInput />);
    await RevalidationAfterInput.play({canvasElement: container});
    await waitFor(() => expect(screen.getByText('password is not same')).toBeInTheDocument())
  })

  test('バリデーションエラー表示後に正しい入力をした場合には、バリデーションエラーが消えている', async () => {
    const { container } = render(<ErrorShouldDisapperWhenValidInput />);
    await ErrorShouldDisapperWhenValidInput.play({canvasElement: container});
    await waitFor(() => expect(screen.queryByText('password is not same')).toBeNull())
  })
})
```
# おわりに
以上、RHFを使った上での個人的なはまりポイントでした。
フォームの厳密な動きや見せ方などを突き詰めていくとかなり難しいなと思いました。
しかし、RHFはかなり奥が深く、提供しているAPIも多数存在しているため、実はうまく組み合わせれば要求仕様を満たせるんじゃないかというワクワク感があってなかなか楽しいです。
まだまだ、細かいところでのエラー表示等、はまりポイントが出たら追加していきたいです。

# はまりポイントではないですが、他のフォームを扱う上でよく使いそうなもの
## ボタンの多連打対策
useFormでは、formStateにisSubmittingという値を返してくれています。
docによると、submit中ならtrueに、それ以外ならfalseになるとのこと
```
true if the form is currently being submitted. false otherwise.
```
なので、これを使用した多連打対策は以下のように、Submit中はbutton要素をdisabledにする みたいにできると思われます。
```tsx: password.tsx
const {
  formState: { errors, isSubmitting },
} = useForm()

<button disabled={isSubmitting}>Submit</button>
```

## 全てのValidationが通った段階でButtonを押せるようにする
基本的にValidationが通っていない時は無闇にボタンをおさせたくないと思うので、disabledにすることが思います。
その際、onBlurで最後の要素のValidationが通った段階でボタンを押せるようにするには、formStateのisValidを使えばすぐにできます。

```tsx: password.tsx
const {
  formState: { errors, isSubmitting, isValid },
} = useForm()

<button disabled={isSubmitting || !isValid}>Submit</button>
```

他にもよくありそうなものとかあったら教えてください！