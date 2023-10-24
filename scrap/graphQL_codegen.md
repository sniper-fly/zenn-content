ディレクトリ構造は以下。

```
├── codegen.ts
├── components
│   ├── Game.tsx
│   ├── Layout.tsx
│   └── QuizBoard.tsx
├── graphql
│   ├── fragment-masking.ts
│   ├── gql.ts
│   ├── graphql.ts
│   └── index.ts
├── next.config.js
├── next-env.d.ts
├── package.json
├── package-lock.json
├── pages
│   ├── api
│   │   └── hello.ts
│   ├── _app.tsx
│   ├── _document.tsx
│   ├── index.tsx
│   └── kon_cards.tsx
├── postcss.config.js
├── public
│   ├── favicon.ico
│   ├── next.svg
│   ├── studio_names.json
│   ├── studios.json
│   └── vercel.svg
├── README.md
├── scripts
│   └── filterAnimeStudio.mjs
├── styles
│   └── globals.css
├── tailwind.config.ts
└── tsconfig.json
```

`codegen.ts`が大事。
内容は以下。

```ts
import { CodegenConfig } from "@graphql-codegen/cli";

const config: CodegenConfig = {
  schema: "https://graphql.anilist.co",
  documents: ["pages/**/*.{ts,tsx}", "components/**/*.{ts,tsx}"],
  generates: {
    "./graphql/": {
      plugins: [],
      preset: "client",
      presetConfig: {
        gqlTagName: "gql",
      },
    },
  },
  ignoreNoDocuments: true,
};

export default config;
```

今回は Anilist API の remote の schema を使いたいので、`schema`に該当の URL を記載する。
`documents` は、GraphQL のクエリが書かれているファイルを指定する。
つまり、今回`documents`に示されたファイルが自動生成の元になる。

`generates`に、自動生成先を指定する。

`package.json`に以下のように script を追加。

```json
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "compile": "graphql-codegen",
    "watch": "graphql-codegen -w"
  },
```

`npm run compile` とすると、

```
├── graphql
│   ├── fragment-masking.ts
│   ├── gql.ts
│   ├── graphql.ts
│   └── index.ts
```

こんな感じにファイルが出来る。
`documents`に書かれているファイル内のクエリが増えたり、編集されたりすると
自動生成されたファイルの中身も都度変わる。
(毎回同じものができるはずだから、これは git 管理しないほうが良いのかな...?)

こういうエラーが出ることがある。

```
✔ Parse Configuration
❯ Generate outputs
  ❯ Generate to ./graphql/
    ✔ Load GraphQL schemas
    ✔ Load GraphQL documents
    ⠹ Generate
[client-preset] the following anonymous operation is skipped:
  query {
    Page(page: 1, perPage: 5) {
      pageInfo {
        total
        currentPage
        lastPage
        hasNextPage
        perPage
      }
      media(search: "けいおん", type: ANIME) {
        id
        title {
          native
        }
        coverImage {
```

これは、

```
query {
```

の箇所に`query`の名前がついていないことが原因なので、該当のファイルを以下のようにテキトウに名前をつけてあげるように編集すれば良い。

```
  query GET_K_ON {
    Page(page: 1, perPage: 5) {
      pageInfo {
        total
        currentPage
        lastPage
        hasNextPage
        perPage
      }
```

ここまでで、Typescript + Next.js + ApolloClient
環境でも取得したクエリのデータから型補完ができる準備は整っている（はず）。

以下のファイルを例にとって使い方を解説すると
```ts
import { useQuery } from "@apollo/client";
import { gql } from "../graphql/gql";

const TRENDING_ANIME = gql(/* GraphQL */ `
  query TRENDING_ANIME {
    Page(page: 1, perPage: 50) {
      media(
        season: FALL
        seasonYear: 2023
        type: ANIME
        format: TV
        sort: POPULARITY_DESC
      ) {
        title {
          native
        }
        coverImage {
          extraLarge
        }
        studios(isMain: true) {
          nodes {
            name
          }
        }
      }
    }
  }
`);

// dataは以下のような構造
// data.Page.media[0].studios.nodes[0].name

const chooseQuiz = (): any => {
  const { loading, error, data } = useQuery(TRENDING_ANIME);
```

![](https://storage.googleapis.com/zenn-user-upload/6a50cd65d023-20231024.png)
vscodeの型補完もきちんと出てますね。
