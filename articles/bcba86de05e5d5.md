---
title: "BunでReact関連の開発環境を構築する"
emoji: "🌊"
type: "tech"
topics:
  - "react"
  - "storybook"
  - "vite"
  - "bun"
  - "vitest"
published: true
published_at: "2022-12-28 19:24"
---

# はじめに
この記事では、Bunを用いてReact周辺の環境構築をしていきます。
今回は以下の構成でいきたいと思います。
- react
- typescript
- vite@4.0.3
- vitest
- react testing library
- storybook

# Reactを立ち上げるまで
## Bunとは
Bunとは、「速くてAll in OneなJavaScriptランタイム」のことです。
Bundler、Transpiler、Package Managerなどが初めから統合されています。  
JSエンジンとしてJavaScriptCoreが導入されていたり、denoやnodeと比較してかなり速いそうです。

詳細は公式docからお願いします。
https://bun.sh/

## ディレクトリ構成
```
.
├── .storybook
│   ├── main.js
│   └── preview.js
├── src
├── bun.lockb
├── index.html
├── package.json
├── vite.config.ts
├── vitest.setup.ts
└── tsconfig.json
```
実際のサンプルコードも載せておきます。
https://github.com/yamakenji24/playground/tree/main/bun-vite-react
## TypeScriptのインストール
package.jsonのdevDependenciesにインストールするには、bunでは`-d`のオプションを使用します。
```shell
bun add -d typescript
```

```json: tsconfig.json
{
  "compilerOptions": {
    "target": "esnext",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "strict": true,
    "forceConsistentCasingInFileNames": true,
    "module": "esnext",
    "moduleResolution": "node",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "preserve",
    "types": ["vitest/globals"]
  },
  "include": ["**/*.ts", "**/*.tsx"],
  "exclude": ["node_modules"]
}

```

## Viteのインストール
2022年12月にViteのv4系がリリースされて触ってみたいのでViteを利用していきます。
ReactのSWC用に新しくプラグインが導入されていたり、Viteそもそものサイズが小さくなっていたりするようです。
https://vitejs.dev/blog/announcing-vite4.html

```shell
bun add -d vite
```

## Reactのインストール
ReactをSWCプラグイン付きで入れていきます。

```shell
bun add react react-dom
bun add -d @types/react @types/react-dom @vitejs/plugin-react-swc
```
react-swcのプラグインを使えるように設定を書きます。
```ts: vite.config.ts
 import { defineConfig } from 'vite';
 import react from '@vitejs/plugin-react-swc'

// https://vitejs.dev/config/
export default defineConfig({
  plugins: [react()]
})
```
爆速で立ち上がると思います。
```shell
 bun run dev
```

::::details Reactを立ち上げる用初期ファイル
```html: index.html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Vite App</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/index.tsx"></script>
  </body>
</html>
```
```tsx: src/index.tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';

const root = ReactDOM.createRoot(document.getElementById('root'));
root.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```
```tsx: src/App.tsx
function App() {
  return (
    <div className="App" role="main">
      <article className="App-article">
        <h3>Welcome to React!</h3>
        <a
          className="App-link"
          href="https://reactjs.org"
          target="_blank"
          rel="noopener noreferrer"
        >
          Learn React
        </a>
      </article>
    </div>
  );
}
export default App;
```
```json: package.json
{
  ...
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build"
  }
}
```
::::

# Test導入
テスト環境として、Vitestを導入してみます。
## Vitestのインストール
```shell
bun add -d vitest
```
vitestをグローバルで使えるように、vite.config.tsに記述していきます。
```diff ts: vite.config.ts
+ /// <reference types="vitest" />
 import { defineConfig } from 'vite';
 import react from '@vitejs/plugin-react-swc'

// https://vitejs.dev/config/
export default defineConfig({
  plugins: [react()],
+ test: {
+   globals: true,
+ },
})
```
なお、vite@4.0.2では、defineConfigからtestがないと型エラーが発生していました。
`4.0.3`に上げることで無事解決されました。
https://github.com/vitest-dev/vitest/issues/2474

適当にテストファイルを書いて走らせてみます。

```src/sample.spec.ts
export const add = (a: number, b: number): number => a + b;
describe("add", () => {
  it("1 + 2 = 3", () => {
    const result = add(1, 2);
    expect(result).toBe(3);
  });
});
```
```diff json: package.json
{
  "scripts": {
+   "test": "vitest",
+   "test:run": "vitest run"
  }
}
```

```shell
~/bun-vite-react (main*) » bun run test   
$ vitest

 DEV  v0.26.2 ~/bun-vite-react

 ✓ src/sample.spec.ts (1)

 Test Files  1 passed (1)
      Tests  1 passed (1)
   Start at  17:20:00
   Duration  5.65s (transform 406ms, setup 1.56s, collect 17ms, tests 3ms)


 PASS  Waiting for file changes...
       press h to show help, press q to quit
```

## React Testing Library
React Testing Libraryを用いてUIのテストも書きます。
```shell
bun add -d @testing-library/jest-dom @testing-library/react @testing-library/user-event jsdom
```

```diff ts: vite.config.ts
export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
+   environment: 'jsdom',
+   setupFiles: './vitest.setup.ts'
  },
})
```

```diff ts: vitest.setup.ts
+ import '@testing-library/jest-dom'
```
お試しにButtonコンポーネントを作成してテストを書いてみます。
```tsx: src/atoms/Button.tsx
type Props = {
  title: string;
  onClick: () => void;
};

export const Button = ({ title, onClick }: Props) => (
  <button onClick={onClick}>{title}</button>
);
```
```tsx: src/atoms/Button.spec.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

import { Button } from './Button';

describe('Button', () => {
  it('render', async () => {
    const mockOnClick = vi.fn();

    const props = {
      onClick: mockOnClick,
      title: 'button title',
    };
    render(<Button {...props} />);

    const button = screen.getByRole('button');
    await userEvent.click(button);

    expect(button).toBeInTheDocument();
    expect(button).toHaveTextContent('button title');
    expect(mockOnClick).toBeCalled();
  });
});
```
```bash
~/bun-vite-react (main*) » bun run test     
$ vitest

 DEV  v0.26.2 ~/bun-vite-react

 ✓ src/sample.spec.ts (1)
 ✓ src/atoms/Button.spec.tsx (1)

 Test Files  2 passed (2)
      Tests  2 passed (2)
   Start at  17:33:16
   Duration  1.05s (transform 234ms, setup 194ms, collect 142ms, tests 52ms)


 PASS  Waiting for file changes...
       press h to show help, press q to quit
```
　
# Storybook導入
最後に、Storybookを導入していきます。
ViteでStorybookを動かしたかったのですが、エラー解決に少し手間がかかりそうでした。
一旦はWebpack5系で動かすようにします。
## Storybookをインストール
```bash
bun add -d @babel/core @storybook/addon-actions @storybook/addon-essentials @storybook/addon-interactions @storybook/addon-links @storybook/builder-webpack5 @storybook/manager-webpack5 @storybook/react @storybook/testing-library
```

Storybook用の設定を書いていきます
```js: main.js
module.exports = {
  stories: ["../src/**/*.stories.mdx", "../src/**/*.stories.@(js|jsx|ts|tsx)"],
  addons: [
    "@storybook/addon-links",
    "@storybook/addon-essentials",
    "@storybook/addon-interactions",
  ],
  framework: "@storybook/react",
  core: {
    builder: {
      name: "webpack5",
      options: {
        lazyCompilation: true,
      },
    },
  },
};
```
お試しにButtonのStoriesを書いてみます
```tsx: src/atoms/Button.stories.tsx
import type { ComponentMeta, ComponentStoryObj } from "@storybook/react";
import { Button } from "./Button";

export default {
  title: "Atoms/Button",
  component: Button,
  args: {
    title: "button",
    onClick: () => {},
  },
} as ComponentMeta<typeof Button>;

export const Default: ComponentStoryObj<typeof Button> = {
  args: {
    title: "Default ボタン！！",
  },
};

export const DifferentText: ComponentStoryObj<typeof Button> = {
  args: {
    title: "違うテキスト",
  },
};
```

`bun run storybook`
```diff json: package.json
  "scripts": {
+   "storybook": "start-storybook -p 6006",
+   "build-storybook": "build-storybook"
  }
```

## おまけ： Viteで頑張って動かしてみる
Storybookをviteで動かすには、vite用のbuilderをインストールする必要があります。
またwebpack関連のライブラリは削除します。
```bash
bun add -d @storybook/builder-vite
bun remove @storybook/builder-webpack5 @storybook/manager-webpack5
```
```diff js: main.js
 module.exports = {
   stories: ["../src/**/*.stories.mdx", "../src/**/*.stories.@(js|jsx|ts|tsx)"],
   framework: "@storybook/react",
   addons: ["@storybook/addon-links", "@storybook/addon-essentials", "@storybook/addon-interactions"],
   core: {
-    builder: {
-      name: "webpack5",
-      options: {
-        lazyCompilation: true,
-      },
-    },
+    builder: '@storybook/builder-vite'
   },
 };
```
ここで走らせてみると、以下のエラーが発生しました。
```bash
TypeError: Cannot read properties of undefined (reading 'createSnapshot')
```
まだissueがオープン状態ですが、以下の状態と類似と思われます。
https://github.com/storybookjs/storybook/issues/18696
以下のリリースから、v7系のαないしはβであれば、動かせるかもしれないということで、現時点最新であるbeta.15を入れていきます。

https://github.com/storybookjs/storybook/pull/19007
https://github.com/storybookjs/storybook/releases/tag/v7.0.0-beta.2

Storybookの7系からChangesがいくつか入っているので、それの対応も行います。
まず、`start-storybook`や`build-storybook`が廃止され、Storybook`s ClIに置き換わっています。
https://github.com/storybookjs/storybook/blob/next/MIGRATION.md#start-storybook--build-storybook-binaries-removed

さらには、`main.js`の`framework`が必須となり、指定のフレームワークを入れる必要があります。
`@storybook/react`ではなく、今回は`@storybook/react-vite`を指定します。
https://github.com/storybookjs/storybook/blob/next/MIGRATION.md#framework-field-mandatory

ここで実行してみると次のエラーが発生しました。
```bash
[vite] Internal server error: Failed to resolve import "@storybook/preview-web" from "../../../../../../virtual:/@storybook/builder-vite/vite-app.js". Does the file exist?
```
ここで一旦お手上げ。