---
title: "AstroのIslandsでUIフレームワーク混ぜて遊んでみた"
emoji: "🚀"
type: "tech"
topics:
  - "react"
  - "vuejs"
  - "tech"
  - "svelte"
  - "astro"
published: true
published_at: "2022-09-11 23:33"
---

# はじめに
先月Astroが発表され、楽にUIフレームワークをミックスできる ということで少し遊んでみました。
構築内容は提示版のようなデモアプリの参考仕様を提示してくれる[RealWorld](https://realworld-docs.netlify.app/docs/intro) を元に、React、Svelte、Vueをミックスして実装していきます。

# Astroとは
https://astro.build/
Astroは高速なWebサイトを構築するための静的サイトビルダーです。
MPAを採用しており、ビルド時に不必要なJavaScriptを削除して生成します。
どういう時にAstroを採用したらいいかなどは[Astroを選ぶ理由](https://docs.astro.build/ja/concepts/why-astro/)の公式docに記述されていますので、興味のある方はそちらをご覧ください。

## Astro Islands
ここでは、Astroの特徴である[Astro Islands](https://docs.astro.build/ja/concepts/islands/)について、簡単に紹介します。
Astro Islandsはアイランドアーキテクチャを採用したWebアーキテクチャです。
アイランドアーキテクチャとは、ページ内でサーバ側でレンダリングされる静的な部分とインタラクティブな部分をそれぞれ独立して表示させる手法です。
独立したコンポーネントごとにハイドレーションを行い、不必要なJavaScriptをを削減してパフォーマンスを向上させます。
Astroでは、それらの独立したコンポーネントごとに使用したいUIフレームワークを選択することができ、フレームワークに依存することなく実装を行うことができるようになっています。

## Nano Stores
ページ内外で状態を共有するために、[Nano Stores](https://github.com/nanostores/nanostores)を今回は使用します。
軽量な状態管理ライブラリで、フレームワークに依存しておらず、フレームワーク間での状態管理がしやすいという点が挙げられます。
また、AstroではPartial Hydrationという明示的にクライアントサイドのJSを定義する必要があり、ReduxやContext ProviderみたいなWrapperは使用できないというのもあります。

利用する時には、各ライブラリをインストールするだけで利用が可能になります。
```
npm install nanostores
```

また、各フレームワークで状態の更新や取得を行いたい際には、それぞれのライブラリをインストールして使用します。
```
npm i nanostores @nanostores/react
```

# いざ混ぜてみる
## 今回の構成
以下のようなUIと仕様を実装していきます。
青い枠で囲んだものはReactで実装するもので、Headerとタグ一覧の表示します。
黄色い枠で囲んだものがSvelteで実装するもので、どの分類の記事一覧を表示するかを管理しています。
緑の枠で囲んだものがVueで実装するもので、記事一覧を表示します。
なお、今回は異なるUIフレームワークのコンポーネント間の実装と状態の共有を取り上げたいと思いますので、詳細な実装については省かせていただきます。

![](https://storage.googleapis.com/zenn-user-upload/85becc5a6b27-20220911.png)

## 各状態管理
ページ間ないしはコンポーネント間で状態を共有するためにNano Storesを使用しています。
例えば、選択したタグを共通で持ちたいとします。
Atomsで定義したものにはstringsやnumber,arrayを格納することができます。ここではタグ一覧と選択したタグを保持します。
格納している状態に対して、何かしらの変更を行う際にはActionsを使用します。
ここまで使ってみて、使い勝手的に[Recoil](https://recoiljs.org/)みたいだなという感想でした。
```ts:tagStore.ts
import { atom, action } from "nanostores";
import { getTagsService } from "../services/get-tags";

export const tags = atom<string[]>([]);
export const tag = atom<string>('');

export const getTags = action(tags, "getTags", async (store) => {
  const newTags = await getTagsService();
  if (newTags) {
    store.set(newTags);
  }
});

```

## 各コンポーネントで状態を更新、取得する
atomsやmapsで格納している状態を各コンポーネントで更新や取得を行います。
@nanostoresの`useStore`や`$store`を用います。それ以外にもget()を用いて取得できますが、値の更新はされず、現在の取得した値のみを取り扱うようです。https://github.com/nanostores/nanostores#reduce-get-usage-outside-of-tests

### タグ一覧コンポーネント (React)
タグ一覧を取得しStoreに値を保持した後、一覧を表示するコンポーネントです。また、タグを選択することでそのタグに分類される記事一覧を取得します。

今回はタグ一覧を取得する処理をuseEffect内に記述していますが、Store側にロジックを移行できるみたいです。https://github.com/nanostores/nanostores#move-logic-from-components-to-stores
選択したタグは本コンポーネントで担当し、選択されたタグでどの記事一覧を表示するかはFeedNavigation並びにArticlePreviewLayoutで行います。

```tsx:TagLayout.tsx
import { useStore } from "@nanostores/react";
import { useEffect, useCallback } from 'react';
import { getTags, tags, tag } from "../../store/tagStore";
import { Tag } from './Tag'

export const TagLayout = () => {
  const tagList = useStore(tags);
  useEffect(() => {
    getTags();
  }, [])

  const handleTagOnClick = useCallback((selectedTag: string) => () => {
    tag.set(selectedTag)
  }, []);

  if (!tagList.length) {
    return <div>Now Loading....</div>
  }

  return (
    <div className="tag-list">
      {tagList.map((tag) => (
        <Tag key={tag} tag={tag} handleTagOnClick={handleTagOnClick(tag)}/>
      ))}
    </div>
  );
};

```

### FeedNavigationコンポーネント (Svelte)
FeedNavigationでは、自分、全体、タグの3つののNavigationがあり、選択することでどの記事一覧を表示するかを切り替えることができます。
タグはタグ一覧から選択されたものを使用しており、nanoStoreでコンポーネント間の状態の共有を行っています。

```ts:FeedNavigation.svelte
<script>
  import { tag } from "../../store/tagStore";
  import { userInfo } from "../../store/userStore";
  import {
    getGlobalArticles,
    getArticlesWithRequirements,
  } from "../../store/articleStore";

  function handleChangeFeedNav(e) {
    const name = e.currentTarget.name;
    switch (name) {
      case "global": {
        getGlobalArticles();
        break;
      }
      case "mine": {
        getArticlesWithRequirements({ author: $userInfo.username });
        break;
      }
      default: {
        getArticlesWithRequirements({ tag: name });
        break;
      }
    }
  }
</script>

<div class="feed-toggle">
  <ul class="nav nav-pills outline-active">
    <li class="nav-item">
      <button on:click={handleChangeFeedNav} name="mine" class="nav-link">
        Your Feed
      </button>
    </li>
    <li class="nav-item">
      <button
        on:click={handleChangeFeedNav}
        name="global"
        class="nav-link"
      >
        Global Feed
      </button>
    </li>
    {#if $tag}
      <li class="nav-item">
        <button on:click={handleChangeFeedNav} name={$tag} class="nav-link">
          {$tag}
        </button>
      </li>
    {/if}
  </ul>
</div>
```

### 記事一覧コンポーネント (Vue)
FeedNavigationでどの記事一覧を表示するかを選択し、一覧を取得しています。
このコンポーネントでは選択したものをStoreから受け取って、そのまま表示します。

```vue:ArticlePreviewLayout.vue
<template>
  <div v-if="articles.length">
    <ArticlePreview
      v-for="article in articles"
      v-bind:key="article.slug"
      :author="article.author.username"
      :date="article.createdAt"
      :favCounts="article.favoritesCount"
      :title="article.title"
      :description="article.description"
    />
  </div>
</template>

<script>
import ArticlePreview from './ArticlePreview.vue';
import { getGlobalArticles, postedArticles } from "../../store/articleStore";
import { useStore } from "@nanostores/vue";

export default {
  components: {
    ArticlePreview,
  },
  created() {
    getGlobalArticles();
  },
  setup() {
    const articles = useStore(postedArticles);
    return { articles };
  },
};
</script>

```

## Astroコンポーネント
作成したコンポーネントをAstroコンポーネントとよばれる、ベースとなるテンプレートコンポーネントに繋ぎ込んでいきます。
Astroコンポーネントでは、デフォルトではビルド時にコンポーネント内部のJSは全て実行され、クライアントサイドのランタイムを持たないHTMLのみのコンポーネントとして表示されます。そのため、インタラクティブなコンポーネントを扱う際には、`client:*`を明示的に指定して、どのようにレンダリング・ハイドレーションされるかを定義する必要があります。
例えば、`client:load`はページ読み込み時にJSのインポートが開始されたり、`client:idle`はページ読み込みが終了し、ブラウザが`requestIdleCallback`をサポートしていたらコンポーネントの読み込みとハイドレートが開始されます。

今回はわかりやすく`client:only="各フレームワーク"`を定義しています。
`client:only={}`はサーバサイドでのレンダリングをスキップし、クライアントのみでレンダリングするようになります。また、ビルド時に生成しないため、どのフレームワークのコンポーネントを利用しているのかが不明であり、各フレームワーク名を正しく渡す必要があります。

```jsx:index.astro
---
import Layout from "../layouts/Layout.astro";
import ArticlePreviewLayout from "../features/ArticlePreview/ArticlePreviewLayout.vue";
import { TagLayout } from "../features/TagLayout";
import FeedNavigation from "../features/FeedNavigation/FeedNavigation.svelte";
---

<Layout title="RealWorld Astro ver">
  <div class="home-page">
    <div class="banner">
      ...
    </div>

    <div class="container page">
      <div class="row">
        <div class="col-md-9">
          <FeedNavigation client:only="svelte" />
          <ArticlePreviewLayout client:only="vue" />
        </div>

        <div class="col-md-3">
          <div class="sidebar">
            <p>Popular Tags</p>
            <TagLayout client:only="react" />
          </div>
        </div>
      </div>
    </div>
  </div>
</Layout>
```

# おわりに
今回は、Astroを用いて複数のUIフレームワークをミックスして遊んでみました。
Astroコンポーネントをベースに状態管理ライブラリであるNano Storesを用いることでコンポーネント間の状態管理も非常に簡単に行うことができました。
ただ、公式にも記述されていることですが、複雑な画面構成のWebアプリを作成しようと思うと少し辛いな、と実装しながら感じました。
これまでReactを用いたSPA開発に慣れてしまっていたというのもあり、今回のMPA形式で開発を行う際にページ間移動時に強制リロードが走ったり、どこまでをページとして管理するかみたいなところがいつもと少し違うなと思いました。
逆に、ページ内部でどのコンポーネントをjsで読み込むか、それに応じて状態をどう持つかなどを考える必要があり、アイランドアーキテクチャの一部を低コストで触ることができました。
インストールするだけで利用することができるので、アイランドアーキテクチャや複数のUIフレームを混ぜて遊んでみたい方はぜひAstro触ってみたらいいなと思いました！
