---
title: "Shopifyアプリの公式チュートリアルをLaravelで実装する"
emoji: "🛍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [shopify,laravel,php]
published: true

---
こんにちは。
shopifyがここ数年でとてつもなく成長しているので気になり、Shopifyアプリを作ってみることにしました。

ドキュメントを見た瞬間「あー天才達が作ったんだなぁ」と思いました。
全然わかりません。

PHPのテンプレートはあるようなので、こちらを利用してチュートリアルやっていきたいと思います。
https://github.com/Shopify/shopify-app-template-php

※ 少し前にやっていたので、現在の公式チュートリアルはRemixで書かれていますが、Expressで書かれていた頃の公式チュートリアルを元にLaravelで書いてます。

# 概要
shopifyアプリってなんですか？らへんはこちらをお読みください。
https://shopify.dev/docs/apps/getting-started

このチュートリアルでは、ユーザーが管理画面でアクセスできる商品のQRコード生成アプリを作成します。
テンプレートの技術スタックについては以下に書いてますが、laravelとviteを利用していますね。
https://github.com/Shopify/shopify-app-template-php#tech-stack

## 前提条件
- パートナーアカウントと開発ストアが作成されている
- Node.js 16 以降がインストールされている。
- Node.js パッケージ マネージャー ( npm、Yarn 1.x、またはpnpm )がインストールされている。（今回はnpmでやります）
- PHP がインストールされている。
- Composer がインストールされている。

# アプリを作成する
## テンプレートのインストール
まずはphpテンプレートのREADMEに従い、新規アプリを作成します。
任意のディレクトリで以下のコマンドを実行します。
```
$ npm init @shopify/app@latest -- --template php
```
以下のようにアプリ名を入力するよう求められるので、アプリ名を入力します。ちなみにshopifyの名前はアプリ名に入れることはできませんでした。
![](/images/shopify_app_templete_install.png)
## Laravel アプリのセットアップ
依存関係やデータベース設定等は自動で生成されないため、手動でセットアップします。 READMEの手順に従って設定していきます。

1. まず、webフォルダに移動します。webフォルダにlaravelアプリが入ってるようです。

```
$ cd web
```
2. Composerの依存関係をインストールします。

```
$ composer install
```
3. .env.exampleファイルをコピーして.envファイルを作成します。
```
$ cp .env.example .env
```
4. storageディレクトリにSQLiteデータベースを作成し、.envにデータベースを設定します。
envファイルのDB_DATABASEに設定するのは作成したdb.sqliteの**フルパス**を設定します。
```
$ touch storage/db.sqlite
```

``` diff:.env
- DB_DATABASE=storage/db.sqlite
+ DB_DATABASE=storage/db.sqliteまでのフルパス
```
5. アプリAPP_KEYを生成します。
```
$ php artisan key:generate
```
6. shopifyと連携するために必要なテーブルを作成します。sessionsというテーブルが出来上がります。
```
$ php artisan migrate
```

これで、Laravel アプリを実行する準備が整いました。これで、アプリのルート フォルダーに戻って続行できます。
```
cd ..
```

# ローカルサーバーを起動する
> アプリの作成後、ローカル開発サーバーとShopify CLIを使用して作業できます。
> Shopify CLIはCloudflareを使用して、HTTPS URLを使用してアプリにアクセスできるようにするトンネルを作成します。

とのことです。
コマンドを叩くと全部やってくれますが、Shopify CLIがCloudflareを使用して、アプリにHTTPS URLでアクセスできるトンネルを作成します。つまり、ローカルで開発中のアプリを外部から安全にアクセスできるようになります。ってことかな？
1. 開発サーバーを立ち上げる
```
$ npm run dev
```
2. shopifyへログイン
以下のようにブラウザからログインを求められます。
![](/images/shopify-login.png)
何かボタンを押すとブラウザのログイン画面が表示されるので、ログイン後、またターミナルに戻ってきます。

3. アプリ名入力とアプリを入れるショップを選ぶ
ターミナルに戻るとこんな画面になっているので、Yes, create it as a new appを選択します。
![](/images/shopify-create-new-app.png)

アプリ名を聞かれるので、アプリ名を入力→アプリを入れるショップ名を聞かれるので、ショップを選択。
Have Shopify automatically update your app's URL in order to create a preview experience?と聞かれたらyesを選択でいいです。

4. アプリ画面へ

![](/images/shopify-preview.png)
無事に立ち上がるとPreview URLが生成されるので、「p」を押してアプリ画面へ遷移します。
Preview URLをブラウザに貼り付けてもいいです。
![](/images/app-install.png)
上記のような画面になっていると思うので、右上のアプリをインストールボタンを押します。
するとショップ画面にアプリが追加され、以下のような画面になります。

テンプレートにはボタンを押すと商品が5つ追加されるアプリが最初から入ってるみたいですね。
![](/images/app-page.png)

ちなみに、開発時はphpのビルトインサーバーで動いてます。
これどうにかdockerで自分専用環境構築できるようにできませんかね？

ngrok等利用して自分で認証処理書けばいいんでしょうが、shopify CLIで自動でトンネルしてくれるの便利なので、そうしたいです、、
誰か知ってたら教えてください。
とりあえず初めてなので、このままいきます。

# QRコードアプリの概要
公式チュートリアルでは、商品のQR コードを作成するアプリを作成します。

QRコードをスキャンすると、ユーザーは商品のチェックアウト画面、または商品ページに移動します。
アプリは QR コードがスキャンされるたびにログを記録し、スキャンされた回数をアプリに表示します。

shopify特有の用語については以下で説明します。
## Polarisについて
shopifyではPolarisというUIコンポーネントを使用します。
これによって Shopifyのアプリ設計ガイドラインに準拠したUIを作成します。
これを使えば、管理画面のUIをこちらで合わせる必要はなさそうです。
https://polaris.shopify.com/components?shpxid=2647e40c-5513-401F-B0CF-3D3408523C09

## Shopify App Bridgeについて
Shopify App BridgeはShopifyストアの管理画面内に自分のアプリを埋め込むためのツールセットです。
一言で説明するのも、理解するのも難しいですが、
Shopify 管理画面の左側にナビゲーションメニューを表示したりすることができるらしいです。
https://shopify.dev/docs/api/app-bridge

# フロントエンドの構築
フロントエンドは/web/frontend配下で管理しており、Reactで実装していきます。
**以前**の公式チュートリアルと同じなので、中身は割愛します。
実装するページは以下です。
- QRコード一覧表示画面
- QRコード新規登録画面
- QRコード編集画面
## ホーム画面の編集
web/frontend/pages/index.jsxを以下のように上書きします。
```jsx:web/frontend/pages/index.jsx
import { useNavigate, TitleBar, Loading } from "@shopify/app-bridge-react";
import {
  Card,
  EmptyState,
  Layout,
  Page,
  SkeletonBodyText,
} from "@shopify/polaris";

export default function HomePage() {
  /*
    Add an App Bridge useNavigate hook to set up the navigate function.
    This function modifies the top-level browser URL so that you can
    navigate within the embedded app and keep the browser in sync on reload.
  */
  const navigate = useNavigate();

  /*
    These are mock values. Setting these values lets you preview the loading markup and the empty state.
  */
  const isLoading = false;
  const isRefetching = false;
  const QRCodes = [];

  /* loadingMarkup uses the loading component from AppBridge and components from Polaris  */
  const loadingMarkup = isLoading ? (
    <Card sectioned>
      <Loading />
      <SkeletonBodyText />
    </Card>
  ) : null;

  /* Use Polaris Card and EmptyState components to define the contents of the empty state */
  const emptyStateMarkup =
    !isLoading && !QRCodes?.length ? (
      <Card sectioned>
        <EmptyState
          heading="Create unique QR codes for your product"
          /* This button will take the user to a Create a QR code page */
          action={{
            content: "Create QR code",
            onAction: () => navigate("/qrcodes/new"),
          }}
          image="https://cdn.shopify.com/s/files/1/0262/4071/2726/files/emptystate-files.png"
        >
          <p>
            Allow customers to scan codes and buy products using their phones.
          </p>
        </EmptyState>
      </Card>
    ) : null;

  /*
    Use Polaris Page and TitleBar components to create the page layout,
    and include the empty state contents set above.
  */
  return (
    <Page>
      <TitleBar
        title="QR codes"
        primaryAction={{
          content: "Create QR code",
          onAction: () => navigate("/qrcodes/new"),
        }}
      />
      <Layout>
        <Layout.Section>
          {loadingMarkup}
          {emptyStateMarkup}
        </Layout.Section>
      </Layout>
    </Page>
  );
}
```

管理画面で以下のようになっていればOKです。
![](/images/qrcode_index.png)

## QRコードの登録フォームを作成
フォームには@shopify/react-formというライブラリを利用するので、インストールします。
```
$ cd web/frontend
$ npm install @shopify/react-form
```
続いて、web/frontend/componentsにQRCodeForm.jsxファイルを作成し、編集します。
```jsx:web/frontend/components/QRCodeForm.jsx
import { useState, useCallback } from "react";
import {
  Banner,
  Card,
  Form,
  FormLayout,
  TextField,
  Button,
  ChoiceList,
  Select,
  Thumbnail,
  Icon,
  Stack,
  TextStyle,
  Layout,
  EmptyState,
} from "@shopify/polaris";
import {
  ContextualSaveBar,
  ResourcePicker,
  useNavigate,
} from "@shopify/app-bridge-react";
import { ImageMajor, AlertMinor } from "@shopify/polaris-icons";

/* Import the useAuthenticatedFetch hook included in the Node app template */
import { useAuthenticatedFetch, useAppQuery } from "../hooks";

/* Import custom hooks for forms */
import { useForm, useField, notEmptyString } from "@shopify/react-form";

const NO_DISCOUNT_OPTION = { label: "No discount", value: "" };

/*
  The discount codes available in the store.

  This variable will only have a value after retrieving discount codes from the API.
*/
const DISCOUNT_CODES = {};

export function QRCodeForm({ QRCode: InitialQRCode }) {
  const [QRCode, setQRCode] = useState(InitialQRCode);
  const [showResourcePicker, setShowResourcePicker] = useState(false);
  const [selectedProduct, setSelectedProduct] = useState(QRCode?.product);
  const navigate = useNavigate();
  const fetch = useAuthenticatedFetch();
  const deletedProduct = QRCode?.product?.title === "Deleted product";


  /*
    This is a placeholder function that is triggered when the user hits the "Save" button.

    It will be replaced by a different function when the frontend is connected to the backend.
  */
  const onSubmit = (body) => console.log("submit", body);

  /*
    Sets up the form state with the useForm hook.

    Accepts a "fields" object that sets up each individual field with a default value and validation rules.

    Returns a "fields" object that is destructured to access each of the fields individually, so they can be used in other parts of the component.

    Returns helpers to manage form state, as well as component state that is based on form state.
  */
  const {
    fields: {
      title,
      productId,
      variantId,
      handle,
      discountId,
      discountCode,
      destination,
    },
    dirty,
    reset,
    submitting,
    submit,
    makeClean,
  } = useForm({
    fields: {
      title: useField({
        value: QRCode?.title || "",
        validates: [notEmptyString("Please name your QR code")],
      }),
      productId: useField({
        value: deletedProduct ? "Deleted product" : QRCode?.product?.id || "",
        validates: [notEmptyString("Please select a product")],
      }),
      variantId: useField(QRCode?.variantId || ""),
      handle: useField(QRCode?.handle || ""),
      destination: useField(
        QRCode?.destination ? [QRCode.destination] : ["product"]
      ),
      discountId: useField(QRCode?.discountId || NO_DISCOUNT_OPTION.value),
      discountCode: useField(QRCode?.discountCode || ""),
    },
    onSubmit,
  });

  const QRCodeURL = QRCode
    ? new URL(`/qrcodes/${QRCode.id}/image`, location.toString()).toString()
    : null;

  /*
    This function is called with the selected product whenever the user clicks "Add" in the ResourcePicker.

    It takes the first item in the selection array and sets the selected product to an object with the properties from the "selection" argument.

    It updates the form state using the "onChange" methods attached to the form fields.

    Finally, closes the ResourcePicker.
  */
  const handleProductChange = useCallback(({ selection }) => {
    setSelectedProduct({
      title: selection[0].title,
      images: selection[0].images,
      handle: selection[0].handle,
    });
    productId.onChange(selection[0].id);
    variantId.onChange(selection[0].variants[0].id);
    handle.onChange(selection[0].handle);
    setShowResourcePicker(false);
  }, []);


  /*
    This function updates the form state whenever a user selects a new discount option.
  */
  const handleDiscountChange = useCallback((id) => {
    discountId.onChange(id);
    discountCode.onChange(DISCOUNT_CODES[id] || "");
  }, []);


  /*
    This function is called when a user clicks "Select product" or cancels the ProductPicker.

    It switches between a show and hide state.
  */
  const toggleResourcePicker = useCallback(
    () => setShowResourcePicker(!showResourcePicker),
    [showResourcePicker]
  );

  /*
    This is a placeholder function that is triggered when the user hits the "Delete" button.

    It will be replaced by a different function when the frontend is connected to the backend.
  */
  const isDeleting = false;
  const deleteQRCode = () => console.log("delete");


  /*
    This array is used in a select field in the form to manage discount options.

    It will be extended when the frontend is connected to the backend and the array is populated with discount data from the store.

    For now, it contains only the default value.
  */
  const shopData = null;
  const isLoadingShopData = true;
  const discountOptions = [NO_DISCOUNT_OPTION];


  /*
    This function runs when a user clicks the "Go to destination" button.

    It uses data from the App Bridge context as well as form state to construct destination URLs using the URL helpers you created.
  */
  const goToDestination = useCallback(() => {
    if (!selectedProduct) return;
    const data = {
      shopUrl: shopData?.shop.url,
      productHandle: handle.value || selectedProduct.handle,
      discountCode: discountCode.value || undefined,
      variantId: variantId.value,
    };

    const targetURL =
      deletedProduct || destination.value[0] === "product"
        ? productViewURL(data)
        : productCheckoutURL(data);

    window.open(targetURL, "_blank", "noreferrer,noopener");
  }, [QRCode, selectedProduct, destination, discountCode, handle, variantId, shopData]);


  /*
    These variables are used to display product images, and will be populated when image URLs can be retrieved from the Admin.
  */
  const imageSrc = selectedProduct?.images?.edges?.[0]?.node?.url;
  const originalImageSrc = selectedProduct?.images?.[0]?.originalSrc;
  const altText =
    selectedProduct?.images?.[0]?.altText || selectedProduct?.title;

  /* The form layout, created using Polaris and App Bridge components. */
  return (
    <Stack vertical>
      {deletedProduct && (
        <Banner
          title="The product for this QR code no longer exists."
          status="critical"
        >
          <p>
            Scans will be directed to a 404 page, or you can choose another
            product for this QR code.
          </p>
        </Banner>
      )}
      <Layout>
        <Layout.Section>
          <Form>
            <ContextualSaveBar
              saveAction={{
                label: "Save",
                onAction: submit,
                loading: submitting,
                disabled: submitting,
              }}
              discardAction={{
                label: "Discard",
                onAction: reset,
                loading: submitting,
                disabled: submitting,
              }}
              visible={dirty}
              fullWidth
            />
            <FormLayout>
              <Card sectioned title="Title">
                <TextField
                  {...title}
                  label="Title"
                  labelHidden
                  helpText="Only store staff can see this title"
                />
              </Card>

              <Card
                title="Product"
                actions={[
                  {
                    content: productId.value
                      ? "Change product"
                      : "Select product",
                    onAction: toggleResourcePicker,
                  },
                ]}
              >
                <Card.Section>
                  {showResourcePicker && (
                    <ResourcePicker
                      resourceType="Product"
                      showVariants={false}
                      selectMultiple={false}
                      onCancel={toggleResourcePicker}
                      onSelection={handleProductChange}
                      open
                    />
                  )}
                  {productId.value ? (
                    <Stack alignment="center">
                      {imageSrc || originalImageSrc ? (
                        <Thumbnail
                          source={imageSrc || originalImageSrc}
                          alt={altText}
                        />
                      ) : (
                        <Thumbnail
                          source={ImageMajor}
                          color="base"
                          size="small"
                        />
                      )}
                      <TextStyle variation="strong">
                        {selectedProduct.title}
                      </TextStyle>
                    </Stack>
                  ) : (
                    <Stack vertical spacing="extraTight">
                      <Button onClick={toggleResourcePicker}>
                        Select product
                      </Button>
                      {productId.error && (
                        <Stack spacing="tight">
                          <Icon source={AlertMinor} color="critical" />
                          <TextStyle variation="negative">
                            {productId.error}
                          </TextStyle>
                        </Stack>
                      )}
                    </Stack>
                  )}
                </Card.Section>
                <Card.Section title="Scan Destination">
                  <ChoiceList
                    title="Scan destination"
                    titleHidden
                    choices={[
                      { label: "Link to product page", value: "product" },
                      {
                        label: "Link to checkout page with product in the cart",
                        value: "checkout",
                      },
                    ]}
                    selected={destination.value}
                    onChange={destination.onChange}
                  />
                </Card.Section>
              </Card>
              <Card
                sectioned
                title="Discount"
                actions={[
                  {
                    content: "Create discount",
                    onAction: () =>
                      navigate(
                        {
                          name: "Discount",
                          resource: {
                            create: true,
                          },
                        },
                        { target: "new" }
                      ),
                  },
                ]}
              >
                <Select
                  label="discount code"
                  options={discountOptions}
                  onChange={handleDiscountChange}
                  value={discountId.value}
                  disabled={isLoadingShopData || shopDataError}
                  labelHidden
                />
              </Card>
            </FormLayout>
          </Form>
        </Layout.Section>
        <Layout.Section secondary>
          <Card sectioned title="QR code">
            {QRCode ? (
              <EmptyState imageContained={true} image={QRCodeURL} />
            ) : (
              <EmptyState>
                <p>Your QR code will appear here after you save.</p>
              </EmptyState>
            )}
            <Stack vertical>
              <Button
                fullWidth
                primary
                download
                url={QRCodeURL}
                disabled={!QRCode || isDeleting}
              >
                Download
              </Button>
              <Button
                fullWidth
                onClick={goToDestination}
                disabled={!selectedProduct || isLoadingShopData}
              >
                Go to destination
              </Button>
            </Stack>
          </Card>
        </Layout.Section>
        <Layout.Section>
          {QRCode?.id && (
            <Button
              outline
              destructive
              onClick={deleteQRCode}
              loading={isDeleting}
            >
              Delete QR code
            </Button>
          )}
        </Layout.Section>
      </Layout>
    </Stack>
  );
}

/* Builds a URL to the selected product */
function productViewURL({ shopUrl, productHandle, discountCode }) {
  const url = new URL(shopUrl);
  const productPath = `/products/${productHandle}`;

  /*
    If a discount is selected, then build a URL to the selected discount that redirects to the selected product: /discount/{code}?redirect=/products/{product}
  */
  if (discountCode) {
    url.pathname = `/discount/${discountCode}`;
    url.searchParams.append("redirect", productPath);
  } else {
    url.pathname = productPath;
  }

  return url.toString();
}

/* Builds a URL to a checkout that contains the selected product */
function productCheckoutURL({ shopUrl, variantId, quantity = 1, discountCode }) {
  const url = new URL(shopUrl);
  const id = variantId.replace(
    /gid:\/\/shopify\/ProductVariant\/([0-9]+)/,
    "$1"
  );

  url.pathname = `/cart/${id}:${quantity}`;

  /* Builds a URL to a checkout that contains the selected product with a discount code applied */
  if (discountCode) {
    url.searchParams.append("discount", discountCode);
  }

  return url.toString();
}

```
QRCodeFormコンポーネントは/web/frontend/components/index.jsでexportします。


```diff js:web/frontend/components/index.js
export { ProductsCard } from "./ProductsCard";
export * from "./providers";
+ export { QRCodeForm } from './QRCodeForm'
```

##  QRコード新規登録・編集画面作成
web/frontend/pages/配下にqrcodesディレクトリを作成します。
qrcodesディレクトリに新規登録・編集画面のファイルを作成します。
- 新規登録: new.jsx
- 編集: [id].jsx

NuxtとかNextのようなルーティングでいい感じですね。
new.jsxを編集します。
```jsx:web/frontend/pages/qrcodes/new.jsx
import { Page } from "@shopify/polaris";
import { TitleBar } from "@shopify/app-bridge-react";
import { QRCodeForm } from "../../components";

export default function ManageCode() {
  const breadcrumbs = [{ content: "QR codes", url: "/" }];

  return (
    <Page>
      <TitleBar
        title="Create new QR code"
        breadcrumbs={breadcrumbs}
        primaryAction={null}
      />
      <QRCodeForm />
    </Page>
  );
}
```
続いて[id].jsx編集
```jsx:web/frontend/pages/qrcodes/[id].jsx
import { Card, Page, Layout, SkeletonBodyText } from "@shopify/polaris";
import { Loading, TitleBar } from "@shopify/app-bridge-react";
import { QRCodeForm } from "../../components";

export default function QRCodeEdit() {
  const breadcrumbs = [{ content: "QR codes", url: "/" }];

  /*
    These are mock values.
    Set isLoading to false to preview the page without loading markup.
  */
  const isLoading = false
  const isRefetching = false
  const QRCode = {
    createdAt: '2022-06-13',
    destination: 'checkout',
    title: 'My first QR code',
    product: {}
  }

  /* Loading action and markup that uses App Bridge and Polaris components */
  if (isLoading || isRefetching) {
    return (
      <Page>
        <TitleBar
          title="Edit QR code"
          breadcrumbs={breadcrumbs}
          primaryAction={null}
        />
        <Loading />
        <Layout>
          <Layout.Section>
            <Card sectioned title="Title">
              <SkeletonBodyText />
            </Card>
            <Card title="Product">
              <Card.Section>
                <SkeletonBodyText lines={1} />
              </Card.Section>
              <Card.Section>
                <SkeletonBodyText lines={3} />
              </Card.Section>
            </Card>
            <Card sectioned title="Discount">
              <SkeletonBodyText lines={2} />
            </Card>
          </Layout.Section>
          <Layout.Section secondary>
            <Card sectioned title="QR code" />
          </Layout.Section>
        </Layout>
      </Page>
    );
  }

  return (
    <Page>
      <TitleBar
        title="Edit QR code"
        breadcrumbs={breadcrumbs}
        primaryAction={null}
      />
      <QRCodeForm QRCode={QRCode} />
    </Page>
  );
}
```

管理画面で「Create QR code」新規登録画面に遷移するか確認します。
![](/images/qrcode_form.png)

## QRコード一覧表示コンポーネントの作成
まず、必要なライブラリをインストールします。
```
npm install @shopify/react-hooks dayjs
```
続いて、web/frontend/components/配下にQRCodeIndex.jsxファイルを新規作成します。

```jsx:web/frontend/components/QRCodeIndex.jsx
import { useNavigate } from "@shopify/app-bridge-react";
import {
  Card,
  Icon,
  IndexTable,
  Stack,
  TextStyle,
  Thumbnail,
  UnstyledLink,
} from "@shopify/polaris";
import { DiamondAlertMajor, ImageMajor } from "@shopify/polaris-icons";

/* useMedia is used to support multiple screen sizes */
import { useMedia } from "@shopify/react-hooks";

/* dayjs is used to capture and format the date a QR code was created or modified */
import dayjs from "dayjs";

/* Markup for small screen sizes (mobile) */
function SmallScreenCard({
  id,
  title,
  product,
  discountCode,
  scans,
  createdAt,
  navigate,
}) {
  return (
    <UnstyledLink onClick={() => navigate(`/qrcodes/${id}`)}>
      <div
        style={{ padding: "0.75rem 1rem", borderBottom: "1px solid #E1E3E5" }}
      >
        <Stack>
          <Stack.Item>
            <Thumbnail
              source={product?.images?.edges[0]?.node?.url || ImageMajor}
              alt="placeholder"
              color="base"
              size="small"
            />
          </Stack.Item>
          <Stack.Item fill>
            <Stack vertical={true}>
              <Stack.Item>
                <p>
                  <TextStyle variation="strong">
                    {truncate(title, 35)}
                  </TextStyle>
                </p>
                <p>{truncate(product?.title, 35)}</p>
                <p>{dayjs(createdAt).format("MMMM D, YYYY")}</p>
              </Stack.Item>
              <div style={{ display: "flex" }}>
                <div style={{ flex: "3" }}>
                  <TextStyle variation="subdued">Discount</TextStyle>
                  <p>{discountCode || "-"}</p>
                </div>
                <div style={{ flex: "2" }}>
                  <TextStyle variation="subdued">Scans</TextStyle>
                  <p>{scans}</p>
                </div>
              </div>
            </Stack>
          </Stack.Item>
        </Stack>
      </div>
    </UnstyledLink>
  );
}

export function QRCodeIndex({ QRCodes, loading }) {
  const navigate = useNavigate();

  /* Check if screen is small */
  const isSmallScreen = useMedia("(max-width: 640px)");

  /* Map over QRCodes for small screen */
  const smallScreenMarkup = QRCodes.map((QRCode) => (
    <SmallScreenCard key={QRCode.id} navigate={navigate} {...QRCode} />
  ));

  const resourceName = {
    singular: "QR code",
    plural: "QR codes",
  };

  const rowMarkup = QRCodes.map(
    ({ id, title, product, discountCode, scans, createdAt }, index) => {
      const deletedProduct = product.title.includes("Deleted product");

      /* The form layout, created using Polaris components. Includes the QR code data set above. */
      return (
        <IndexTable.Row
          id={id}
          key={id}
          position={index}
          onClick={() => {
            navigate(`/qrcodes/${id}`);
          }}
        >
          <IndexTable.Cell>
            <Thumbnail
              source={product?.images?.edges[0]?.node?.url || ImageMajor}
              alt="placeholder"
              color="base"
              size="small"
            />
          </IndexTable.Cell>
          <IndexTable.Cell>
            <UnstyledLink data-primary-link url={`/qrcodes/${id}`}>
              {truncate(title, 25)}
            </UnstyledLink>
          </IndexTable.Cell>
          <IndexTable.Cell>
            <Stack>
              {deletedProduct && (
                <Icon source={DiamondAlertMajor} color="critical" />
              )}
              <TextStyle variation={deletedProduct ? "negative" : null}>
                {truncate(product?.title, 25)}
              </TextStyle>
            </Stack>
          </IndexTable.Cell>
          <IndexTable.Cell>{discountCode}</IndexTable.Cell>
          <IndexTable.Cell>
            {dayjs(createdAt).format("MMMM D, YYYY")}
          </IndexTable.Cell>
          <IndexTable.Cell>{scans}</IndexTable.Cell>
        </IndexTable.Row>
      );
    }
  );

  /* A layout for small screens, built using Polaris components */
  return (
    <Card>
      {isSmallScreen ? (
        smallScreenMarkup
      ) : (
        <IndexTable
          resourceName={resourceName}
          itemCount={QRCodes.length}
          headings={[
            { title: "Thumbnail", hidden: true },
            { title: "Title" },
            { title: "Product" },
            { title: "Discount" },
            { title: "Date created" },
            { title: "Scans" },
          ]}
          selectable={false}
          loading={loading}
        >
          {rowMarkup}
        </IndexTable>
      )}
    </Card>
  );
}

/* A function to truncate long strings */
function truncate(str, n) {
  return str.length > n ? str.substr(0, n - 1) + "…" : str;
}
```

web/frontend/components/index.jsでQRCodeIndexをexportします。
```diff js:web/frontend/components/index.js
export { ProductsCard } from "./ProductsCard";
export * from "./providers";
export { QRCodeForm } from './QRCodeForm';
+ export { QRCodeIndex } from "./QRCodeIndex";
```
最後に、サンプルのモックデータを作成して、一覧表示してみます。
```jsx:web/frontend/pages/index.jsx
import { useNavigate, TitleBar, Loading } from "@shopify/app-bridge-react";
import {
  Card,
  EmptyState,
  Layout,
  Page,
  SkeletonBodyText,
} from "@shopify/polaris";
import { QRCodeIndex } from "../components";

export default function HomePage() {
  /*
    Add an App Bridge useNavigate hook to set up the navigate function.
    This function modifies the top-level browser URL so that you can
    navigate within the embedded app and keep the browser in sync on reload.
  */
  const navigate = useNavigate();

  /*
    These are mock values. Setting these values lets you preview the loading markup and the empty state.
  */
  const isLoading = false;
  const isRefetching = false;
  const QRCodes = [
    {
      createdAt: "2022-06-13",
      destination: "checkout",
      title: "My first QR code",
      id: 1,
      discountCode: "SUMMERDISCOUNT",
      product: {
        title: "Faded t-shirt",
      }
    },
    {
      createdAt: "2022-06-13",
      destination: "product",
      title: "My second QR code",
      id: 2,
      discountCode: "WINTERDISCOUNT",
      product: {
        title: "Cozy parka",
      }
    },
    {
      createdAt: "2022-06-13",
      destination: "product",
      title: "QR code for deleted product",
      id: 3,
      product: {
        title: "Deleted product",
      }
    },
  ];

  /* Set the QR codes to use in the list */
  const qrCodesMarkup = QRCodes?.length ? (
    <QRCodeIndex QRCodes={QRCodes} loading={isRefetching} />
  ) : null;

  /* loadingMarkup uses the loading component from AppBridge and components from Polaris  */
  const loadingMarkup = isLoading ? (
    <Card sectioned>
      <Loading />
      <SkeletonBodyText />
    </Card>
  ) : null;

  /* Use Polaris Card and EmptyState components to define the contents of the empty state */
  const emptyStateMarkup =
    !isLoading && !QRCodes?.length ? (
      <Card sectioned>
        <EmptyState
          heading="Create unique QR codes for your product"
          action={{
            content: "Create QR code",
            onAction: () => navigate("/qrcodes/new"),
          }}
          image="https://cdn.shopify.com/s/files/1/0262/4071/2726/files/emptystate-files.png"
        >
          <p>
            Allow customers to scan codes and buy products using their phones.
          </p>
        </EmptyState>
      </Card>
    ) : null;

  /*
    Use Polaris Page and TitleBar components to create the page layout,
    and include the empty state contents set above.
  */
  return (
    <Page fullWidth={!!qrCodesMarkup}>
      <TitleBar
        title="QR codes"
        primaryAction={{
          content: "Create QR code",
          onAction: () => navigate("/qrcodes/new"),
        }}
      />
      <Layout>
        <Layout.Section>
          {loadingMarkup}
          {qrCodesMarkup}
          {emptyStateMarkup}
        </Layout.Section>
      </Layout>
    </Page>
  );
}
```

確認
![](/images/qrcode_list.png)

以上でフロントエンドの構築は終わりです！
本番はここからです。
Laravelでバックエンドを構築していきます。

# バックエンドの構築
このQRコードアプリでは、データベースを作成しQRコードのCRUD処理をしていきます。
公式チュートリアルでは、フロントエンド構築→バックエンド構築→最後にバックエンドとフロントエンドの接続という流れでやってましたが、わかりづらかったので、処理を一つずつ実装していこうと思います。

# データベーステーブルの作成
## カラムの内容
id: テーブルの主キー。
title: アプリのユーザーが指定した QR コードの名前。
shopDomain：QRコードを所有する店舗。
productId：QRコードが該当する商品。
handle：QRコードのリンク先URLを作成するために使用します。
variantId：QRコードのリンク先URLを作成するために使用します。
discountId : QRコードのリンク先URLを作成するために使用します。
discountCode : QRコードのリンク先URLを作成するために使用します。
destination：QRコードの送信先です。
scans：QRコードを読み取った回数。

## qr_codesテーブルの作成
以下のコマンドでマイグレーションファイルを作成します。
```
$ cd web
$ php artisan make:migration create_qr_codes_table --create=qr_codes
```

作成されたマイグレーションファイルを編集します。
```php: web/database/migrations/create_qr_codes_table.php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

class CreateQrCodesTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('qr_codes', function (Blueprint $table) {
            $table->id();
            $table->string('shopDomain');
            $table->string('title');
            $table->string('productId');
            $table->string('variantId');
            $table->string('handle');
            $table->string('discountId')->nullable();
            $table->string('discountCode')->nullable();
            $table->string('destination');
            $table->integer('scans')->nullable();
            $table->timestamps();
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::dropIfExists('qr_codes');
    }
}
```
discountIdなどはチュートリアルではnullを許容してませんでしたが、フォームから空送信するとエラーが起きるので、null許容してます。
varcharの長さも指定があったと思いますが、sqliteなんで意味ないかと思ってつけてません。

マイグレーションを実行します。
```
$ php artisan migrate
```

## モデルの作成
テーブルが作成できたので、ついでにモデルも作成しちゃいます。
web/app/Models配下にQrCode.phpファイルを新規作成します。
```php: web/app/Models/QrCode.php
<?php

namespace App\Models;
use Illuminate\Database\Eloquent\Model;

class QrCode extends Model
{
    protected $table = 'qr_codes';
    protected $fillable = [
        'shopDomain',
        'title',
        'productId',
        'variantId',
        'handle',
        'discountId',
        'discountCode',
        'destination',
        'scans'
    ];

    protected $dates = [
        'updated_at',
        'created_at'
    ];
}
```

# 入力フォームからQRコード情報をデータベースに登録する
まず、QRCodeForm.jsxのonSubmitメソッドを編集します。
```diff jsx:web/frontend/components/QRCodeForm.jsx
 import { ImageMajor, AlertMinor } from "@shopify/polaris-icons";
 
 /* Import the useAuthenticatedFetch hook included in the Node app template */
-import { useAuthenticatedFetch, useAppQuery } from "../hooks";
+import { useAuthenticatedFetch, useAppQuery, useCsrf } from "../hooks";
 
 /* Import custom hooks for forms */
 import { useForm, useField, notEmptyString } from "@shopify/react-form";

...

   const navigate = useNavigate();
   const fetch = useAuthenticatedFetch();
   const deletedProduct = QRCode?.product?.title === "Deleted product";
+  const csrf = useCsrf();
 
   /*
     This is a placeholder function that is triggered when the user hits the "Save" button.
 
     It will be replaced by a different function when the frontend is connected to the backend.
   */
-  const onSubmit = (body) => console.log("submit", body);
+  const onSubmit = useCallback(
+    (body) => {
+      (async () => {
+        const parsedBody = body;
+        parsedBody.destination = parsedBody.destination[0];
+        const QRCodeId = QRCode?.id;
+        /* construct the appropriate URL to send the API request to based on whether the QR code is new or being updated */
+        const url = QRCodeId ? `/api/qrcodes/${QRCodeId}` : "/api/qrcodes";
+        /* a condition to select the appropriate HTTP method: PATCH to update a QR code or POST to create a new QR code */
+        const method = QRCodeId ? "PATCH" : "POST";
+        /* use (authenticated) fetch from App Bridge to send the request to the API and, if successful, clear the form to reset the ContextualSaveBar and parse the response JSON */
+        const csrfToken = await csrf();
+        const response = await fetch(url, {
+          method,
+          body: JSON.stringify(parsedBody),
+          headers: {
+            "Content-Type": "application/json",
+            'X-CSRF-TOKEN': csrfToken
+          }
+        });
+        if (response.ok) {
+          makeClean();
+          const QRCode = await response.json();
+          /* if this is a new QR code, then save the QR code and navigate to the edit page; this behavior is the standard when saving resources in the Shopify admin */
+          if (!QRCodeId) {
+            navigate(`/qrcodes/${QRCode.id}`);
+            /* if this is a QR code update, update the QR code state in this component */
+          } else {
+            setQRCode(QRCode);
+          }
+        }
+      })();
+      return { status: "success" };
+    },
+    [QRCode, setQRCode]
+  );
```

useCsrfというメソッドですが、GET以外の通信はCSRF保護しないとエラーとなるため、POST送信する前にCSRFトークンを取得し、リクエストヘッダに含めることで解決することにします。useCsrfの実装は後ほど説明します。

## routeの追加
qrコードデータの送信とcsrfトークンの生成処理をweb/routes/web.phpに追加します。
```php: web/routes/web.php
use App\Http\Controllers\QrCodeController;
...

Route::middleware('shopify.auth')->group(function () {
    Route::get('/api/csrf-token', fn () => ['csrf_token' => csrf_token()]);
    Route::post('/api/qrcodes', [QrCodeController::class, 'create']);
});
```

## csrfトークンの取得
useCsrfというメソッドを作成し、フロントからcsrfトークンを取得できるようにします。
このへん皆さんどうしてますでしょうか？
何か他にいい方法あるよって人がいたら共有して欲しいです。

とりあえず、web/frontend/hooks配下にuseCsrf.jsファイルを新規作成します。
```js:web/frontend/hooks/useCsrf.js
import { useAuthenticatedFetch } from './useAuthenticatedFetch'

export const useCsrf = () => {
  const fetch = useAuthenticatedFetch()

  return async () => {
    const response = await fetch('/api/csrf-token')
    const { csrf_token } = await response.json()

    return csrf_token
  }
}
```
上記をexportします。
web/frontend/hooks/index.jsに以下を追加します。
```diff js:web/frontend/hooks/index.js
export { useAppQuery } from "./useAppQuery";
export { useAuthenticatedFetch } from "./useAuthenticatedFetch";
+ export { useCsrf } from "./useCsrf";
```
あとはクロスサイトリクエストでもクッキーを送信し、Secure属性でHTTPS経由でのみ送信するように設定します。
web/config/session.phpを以下のように編集します。
```diff php:web/config/session.php
...

-    'secure' => env('SESSION_SECURE_COOKIE'),
+    'secure' => true,

...

-    'same_site' => 'lax',
+    'same_site' => 'none',

```
これでなんとかPOSTできるようになりました、、

## 登録処理
shopのドメインを登録したいので、sessionに含まれているshopifyのshopドメインをミドルウェアでリクエスト属性に含めちゃいます。
他のメソッドでも利用しそうだったので。
EnsureShopifySession.phpファイルを編集します。
```diff php:web/app/Http/Middleware/EnsureShopifySession.php
public function handle(Request $request, Closure $next, string $accessMode = self::ACCESS_MODE_OFFLINE)
    {
        switch ($accessMode) {
            case self::ACCESS_MODE_ONLINE:
                $isOnline = true;
                break;
            case self::ACCESS_MODE_OFFLINE:
                $isOnline = false;
                break;
            default:
                throw new Exception(
                    "Unrecognized access mode '$accessMode', accepted values are 'online' and 'offline'"
                );
        }

        $shop = Utils::sanitizeShopDomain($request->query('shop', ''));
        $session = Utils::loadCurrentSession($request->header(), $request->cookie(), $isOnline);

        if ($session && $shop && $session->getShop() !== $shop) {
            // This request is for a different shop. Go straight to login
            return AuthRedirection::redirect($request);
        }

        if ($session && $session->isValid()) {
            if (Config::get('shopify.billing.required')) {
                // The request to check billing status serves to validate that the access token is still valid.
                try {
                    list($hasPayment, $confirmationUrl) =
                        EnsureBilling::check($session, Config::get('shopify.billing'));
                    $proceed = true;

                    if (!$hasPayment) {
                        return TopLevelRedirection::redirect($request, $confirmationUrl);
                    }
                } catch (ShopifyBillingException $e) {
                    $proceed = false;
                }
            } else {
                // Make a request to ensure the access token is still valid. Otherwise, re-authenticate the user.
                $client = new Graphql($session->getShop(), $session->getAccessToken());
                $response = $client->query(self::TEST_GRAPHQL_QUERY);

                $proceed = $response->getStatusCode() === 200;
            }

            if ($proceed) {
                $request->attributes->set('shopifySession', $session);
+               $request->attributes->set('shopDomain', 'https://'.$session->getShop());
                return $next($request);
            }
        }
```
続いて,
チュートリアルではレスポンスをフォーマットして返すメソッドがあったので、そちらを実装します。
web/app/Lib配下にQrCodeHelper.php作成します。
formatQrCodeResponseメソッドでは、レスポンスデータのフォーマット以外に、shopifyが持つ商品データとqrcodesテーブルの整合性を保つという役割もしてますね。
```php:web/app/Lib/QrCodeHelper.php
<?php

namespace App\Lib;

use Illuminate\Http\Request;
use Shopify\Clients\Graphql;
use App\Models\QrCode;
use Exception;

class QrCodeHelper
{
    protected $request;

    private const QR_CODE_ADMIN_QUERY = <<<'QUERY'
    query nodes($ids: [ID!]!) {
        nodes(ids: $ids) {
            ... on Product {
                id
                handle
                title
                images(first: 1) {
                    edges {
                        node {
                            url
                        }
                    }
                }
            }
            ... on ProductVariant {
                id
            }
            ... on DiscountCodeNode {
                id
            }
        }
    }
    QUERY;

    public function __construct(Request $request)
    {
        $this->request = $request;
    }

    public function getQrCodeOr404($id, $checkDomain = true)
    {
        $session = $this->request->get('shopifySession');
        try {
            $response = QrCode::find($id);

            if ($response === null || ($checkDomain && 'https://'.$session->getShop() !== $response->shopDomain)) {
                throw new Exception();
            }

            return $response;

        } catch (\Exception $e) {
            throw new \Exception($e->getMessage(), 500);
        }
    }

    public function formatQrCodeResponse($rawCodeData)
    {
        $session = $this->request->get('shopifySession');

        $ids = [];

        foreach ($rawCodeData as $qrCode) {
            $ids[] = $qrCode['productId'];
            $ids[] = $qrCode['variantId'];

            if ($qrCode['discountId']) {
                $ids[] = $qrCode['discountId'];
            }
        }

        $client = new Graphql($session->getShop(), $session->getAccessToken());

        $adminData = $client->query([
            'query' => self::QR_CODE_ADMIN_QUERY,
            'variables' => ['ids' => $ids],
        ])->getDecodedBody();

        $formattedData = [];

        foreach ($rawCodeData as $qrCode) {
            $product = null;

            foreach ($adminData['data']['nodes'] as $node) {
                if ($qrCode['productId'] === $node['id']) {
                    $product = $node;
                    break;
                }
            }

            if (!$product) {
                $product = ['title' => 'Deleted product'];
            }

            $discountDeleted = $qrCode['discountId'] && !in_array($qrCode['discountId'], array_column($adminData['data']['nodes'], 'id'));

            if ($discountDeleted) {
                QrCode::where('id', $qrCode['id'])
                ->update([
                    'discountId' => '',
                    'discountCode' => '',
                ]);
            }

            $formattedQRCode = array_merge($qrCode, [
                'product' => $product,
                'discountCode' => $discountDeleted ? '' : $qrCode['discountCode'],
            ]);

            unset($formattedQRCode['productId']);

            $formattedData[] = $formattedQRCode;
        }

        return $formattedData;
    }
}
```
あとはControllerを作成し、実際にデータベースに登録する処理を書きます。
```php:web/app/Http/Controllers/QrCodeController.php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use App\Models\QrCode;
use Illuminate\Support\Facades\Log;
use App\Lib\QrCodeHelper;

class QrCodeController extends Controller
{
    private $qrCodeHelper;

    public function __construct(QrCodeHelper $qrCodeHelper)
    {
        $this->qrCodeHelper = $qrCodeHelper;
    }

    /**
     * Create a new QR code record.
     *
     * @param  Request  $request
     * @return \Illuminate\Http\JsonResponse
     */
    public function create (Request $request)
    {
        $qrCodes = $request->all();
        $qrCodes['shopDomain'] = $request->get('shopDomain');
        $qrCodes['scans'] = null;

        $response = $code = null;
        try {
            $rowCodeData = [];
            $code = 201;
            $rowCodeData[] = QrCode::create($qrCodes)->toArray();
            $response = $this->qrCodeHelper->formatQrCodeResponse($rowCodeData)[0];

        } catch (\Exception $e) {
            $code = 500;
            $response = $e->getMessage();

            Log::error("Failed to create qrcodes: $response");

        } finally {
            return response()->json($response, $code);
        }
    }
}
```
いやーテンプレート使ってるのに、csrf対策部分はどこにも記載ないし、自分が間違ってるんでしょうか？
とても不安なので、誰か教えて欲しいです。
まぁとりあえず登録はできました。

# 一覧画面に登録したQRコード情報を表示
フロントエンドを実装した際にモックデータを作成して、表示していた部分を取得データに書き換えます。
web/frontend/pages/index.jsxを編集します。
```diff jsx:web/frontend/pages/index.jsx
...
  SkeletonBodyText,
} from "@shopify/polaris";
import { QRCodeIndex } from "../components";
+ import { useAppQuery } from "../hooks";

export default function HomePage() {
  /*
	
...

-  /*
-    These are mock values. Setting these values lets you preview the loading markup and the empty state.
-  */
-  const isLoading = false;
-  const isRefetching = false;
-  const QRCodes = [
-    {
-      createdAt: "2022-06-13",
-      destination: "checkout",
-      title: "My first QR code",
-      id: 1,
-      discountCode: "SUMMERDISCOUNT",
-      product: {
-        title: "Faded t-shirt",
-      }
-    },
-    {
-      createdAt: "2022-06-13",
-      destination: "product",
-      title: "My second QR code",
-      id: 2,
-      discountCode: "WINTERDISCOUNT",
-      product: {
-        title: "Cozy parka",
-      }
-    },
-    {
-      createdAt: "2022-06-13",
-      destination: "product",
-      title: "QR code for deleted product",
-      id: 3,
-      product: {
-        title: "Deleted product",
-      }
-    },
-  ];
+  /* useAppQuery wraps react-query and the App Bridge authenticatedFetch function */
+  const {
+    data: QRCodes,
+    isLoading,
+
+    /*
+      react-query provides stale-while-revalidate caching.
+      By passing isRefetching to Index Tables we can show stale data and a loading state.
+      Once the query refetches, IndexTable updates and the loading state is removed.
+      This ensures a performant UX.
+    */
+    isRefetching,
+  } = useAppQuery({
+    url: "/api/qrcodes",
+  });
```

useAppQueryというメソッドがテンプレートで既に用意されており、中身を見てみるとshopify のApp Bridgeを利用して、認証が必要なAPIエンドポイントからデータをフェッチし、キャッシュして利用するみたいな感じでした。

続いてrouteとcontrollerを実装します。
```diff php:web/routes/web.php
Route::middleware('shopify.auth')->group(function () {
    Route::get('/api/csrf-token', fn () => ['csrf_token' => csrf_token()]);
    Route::post('/api/qrcodes', [QrCodeController::class, 'create']);
+   Route::get('/api/qrcodes', [QrCodeController::class, 'index']);
});
```

web/app/Http/Controllers/QrCodeController.phpに以下のコードを追加します。
```php:web/app/Http/Controllers/QrCodeController.php
/**
 * get List QR code record.
 *
 * @param  Request  $request
 * @return \Illuminate\Http\JsonResponse
 */
public function index (Request $request)
{
    $response = $code = null;
    try {
        $qrCodes = QrCode::where('shopDomain', $request->get('shopDomain'))->get()->toArray();
        $responseData = [];
        $code = 200;
        foreach ($qrCodes as $qrCode) {
            $responseData[] = $this->__addImageUrl($qrCode);
        }

        $response = $this->qrCodeHelper->formatQrCodeResponse($responseData);

    } catch (\Exception $e) {
        $code = 500;
        $response = $e->getMessage();

        Log::error("Failed to index qrcodes: $response");

    } finally {
        return response()->json($response, $code);
    }
}

private function __addImageUrl($qrCode)
{
    try {
        $qrCode['imageUrl'] = $this->__generateQrcodeImageUrl($qrCode);
    } catch (\Exception $e) {
        Log::error($e->getMessage());
    }

    return $qrCode;
}
    }
```

# QRコード情報の編集
## QRコード情報の取得
QRコード編集画面でもモックデータを表示していたので、こちらも実データにしていきます。
```diff jsx:web/frontend/pages/qrcodes/[id].jsx
 import { Card, Page, Layout, SkeletonBodyText } from "@shopify/polaris";
 import { Loading, TitleBar } from "@shopify/app-bridge-react";
 import { QRCodeForm } from "../../components";
+import { useParams } from "react-router-dom";
+import { useAppQuery } from "../../hooks";
 
 export default function QRCodeEdit() {
   const breadcrumbs = [{ content: "QR codes", url: "/" }];
 
+  const { id } = useParams();
   /*
-    These are mock values.
-    Set isLoading to false to preview the page without loading markup.
+    Fetch the QR code.
+    useAppQuery uses useAuthenticatedQuery from App Bridge to authenticate the request.
+    The backend supplements app data with data queried from the Shopify GraphQL Admin API.
   */
-  const isLoading = false
-  const isRefetching = false
-  const QRCode = {
-    createdAt: '2022-06-13',
-    destination: 'checkout',
-    title: 'My first QR code',
-    product: {}
-  }
+  const {
+    data: QRCode,
+    isLoading,
+    isRefetching,
+  } = useAppQuery({
+    url: `/api/qrcodes/${id}`,
+    reactQueryOptions: {
+      /* Disable refetching because the QRCodeForm component ignores changes to its props */
+      refetchOnReconnect: false,
+    },
+  });

```
```diff php:web/routes/web.php
Route::middleware('shopify.auth')->group(function () {
    Route::get('/api/csrf-token', fn () => ['csrf_token' => csrf_token()]);
    Route::post('/api/qrcodes', [QrCodeController::class, 'create']);
    Route::get('/api/qrcodes', [QrCodeController::class, 'index']);
+   Route::get('/api/qrcodes/{id}', [QrCodeController::class, 'show']);
});
```
QrCodeController.phpに以下のメソッドを追加
```php:web/app/Http/Controllers/QrCodeController.php
/**
 * get QR code record by id.
 *
 * @param  int $id
 * @return \Illuminate\Http\JsonResponse
 */
public function show ($id)
{
    $qrCode = $this->qrCodeHelper->getQrCodeOr404($id);
    if ($qrCode) {
        $code = 200;
        $this->__addImageUrl($qrCode);
        $response = $this->qrCodeHelper->formatQrCodeResponse([$qrCode->toArray()])[0];
        return response()->json($response, $code);
    }
}
```
## QRコード情報のアップデート
フロントエンド側は登録処理の際に実装済みなので、routeとcontrollerのみ実装します。
```diff php:web/routes/web.php
Route::middleware('shopify.auth')->group(function () {
    Route::get('/api/csrf-token', fn () => ['csrf_token' => csrf_token()]);
    Route::post('/api/qrcodes', [QrCodeController::class, 'create']);
    Route::get('/api/qrcodes', [QrCodeController::class, 'index']);
    Route::get('/api/qrcodes/{id}', [QrCodeController::class, 'show']);
+   Route::patch('/api/qrcodes/{id}', [QrCodeController::class, 'update']);
});
```
QrCodeController.phpに以下のメソッドを追加
```php:web/app/Http/Controllers/QrCodeController.php
/**
 * update QR code record.
 *
 * @param  Request  $request
 * @return \Illuminate\Http\JsonResponse
 */
public function update (Request $request, $id)
{
    $response = $code = null;
    $qrCode = $request->all();
    $qrCode['shopDomain'] = $request->get('shopDomain');

    try{
        $rowCodeData = [];
        $code = 200;
        QrCode::where('id', $id)->update($qrCode);

        $rowCodeData[] = QrCode::find($id)->toArray();
        $response = $this->qrCodeHelper->formatQrCodeResponse($rowCodeData)[0];

    } catch (\Exception $e) {
        $code = 500;
        $response = $e->getMessage();

        Log::error("Failed to update qrcodes: $response");

    } finally {
        return response()->json($response, $code);
    }
}
```
# QRコード情報の削除
削除も今までと同じ感じですね。
web/frontend/components/QRCodeForm.jsxのdeleteQRCode部分を変更します。
```diff jsx:web/frontend/components/QRCodeForm.jsx
-  const isDeleting = false;
-  const deleteQRCode = () => console.log("delete");
+  const [isDeleting, setIsDeleting] = useState(false);
+  const deleteQRCode = useCallback(async () => {
+    reset();
+    /* The isDeleting state disables the download button and the delete QR code button to show the user that an action is in progress */
+    setIsDeleting(true);
+    const csrfToken = await csrf();
+    const response = await fetch(`/api/qrcodes/${QRCode.id}`, {
+      method: "DELETE",
+      headers: {
+        "Content-Type": "application/json",
+        'X-CSRF-TOKEN': csrfToken
+      },
+    });
+
+    if (response.ok) {
+      navigate(`/`);
+    }
+  }, [QRCode]);
```
route編集
```diff php:web/routes/web.php
Route::middleware('shopify.auth')->group(function () {
    Route::get('/api/csrf-token', fn () => ['csrf_token' => csrf_token()]);
    Route::post('/api/qrcodes', [QrCodeController::class, 'create']);
    Route::get('/api/qrcodes', [QrCodeController::class, 'index']);
    Route::get('/api/qrcodes/{id}', [QrCodeController::class, 'show']);
    Route::patch('/api/qrcodes/{id}', [QrCodeController::class, 'update']);
+   Route::delete('/api/qrcodes/{id}', [QrCodeController::class, 'delete']);
});
```
QrCodeController.phpに以下のメソッドを追加
```php:web/app/Http/Controllers/QrCodeController.php
/**
 * delete QR code record.
 *
 * @param  int $id
 * @return \Illuminate\Http\JsonResponse
 */
public function delete ($id)
{
    $qrCode = $this->qrCodeHelper->getQrCodeOr404($id);
    if ($qrCode) {
        $code = 200;
        $qrCode->delete();
        return response()->json('', $code);
    }
}
```
# Discount情報の取得
QRコードの入力フォームにDiscount項目があるので、このセレクトボックスを実装します。
ShopifyのAPIを利用して、ディスカウントを取得しますが、アクセススコープがあり、ディスカウントの読み取りを許可しなければなりません。
詳しくはこちら。
https://shopify.dev/docs/api/usage/access-scopes

## アクセススコープに権限を追加
shopify.app.tomlファイルにread_discountsの権限を追加します。
```diff
[access_scopes]
# Learn more at https://shopify.dev/docs/apps/tools/cli/configuration#access_scopes
- scopes = "write_products"
+ scopes = "write_products,read_discounts"
```
続いて、以下のコマンドでアプリの設定を更新します。
```
$ npm run shopify app config push
```
立ち上げ直します。
```
$ npm run dev
```
するとアプリを更新する画面が出てくるので、更新するボタンを押下します。
上記手順でディスカウント読み取りのアクセスを許可することができます。

## 取得処理の実装
web/frontend/components/QRCodeForm.jsxのshopDataを編集します。
```diff jsx:web/frontend/components/QRCodeForm.jsx
-  const shopData = null;
-  const isLoadingShopData = true;
-  const discountOptions = [NO_DISCOUNT_OPTION];
+  const {
+    data: shopData,
+    isLoading: isLoadingShopData,
+    isError: shopDataError,
+    /* useAppQuery makes a query to `/api/shop-data`, which the backend authenticates before fetching the data from the Shopify GraphQL Admin API */
+  } = useAppQuery({ url: "/api/shop-data" });
 
+  /*
+    This array is used in a select field in the form to manage discount options
+  */
+  const discountOptions = shopData
+    ? [
+        NO_DISCOUNT_OPTION,
+        ...shopData.codeDiscountNodes.edges.map(
+          ({ node: { id, codeDiscount } }) => {
+            DISCOUNT_CODES[id] = codeDiscount.codes.edges[0].node.code;
+
+            return {
+              label: codeDiscount.codes.edges[0].node.code,
+              value: id,
+            };
+          }
+        ),
+      ]
+    : [];
```
routeに以下を追加します。
```diff php:web/routes/web.php
Route::middleware('shopify.auth')->group(function () {
    Route::get('/api/csrf-token', fn () => ['csrf_token' => csrf_token()]);
    Route::post('/api/qrcodes', [QrCodeController::class, 'create']);
    Route::get('/api/qrcodes', [QrCodeController::class, 'index']);
    Route::get('/api/qrcodes/{id}', [QrCodeController::class, 'show']);
    Route::patch('/api/qrcodes/{id}', [QrCodeController::class, 'update']);
    Route::delete('/api/qrcodes/{id}', [QrCodeController::class, 'delete']);
+   Route::get('/api/shop-data', [QrCodeController::class, 'getDisCounts']);
});
```
controllerを編集します。
```diff php:web/app/Http/Controllers/QrCodeController.php
 use App\Models\QrCode;
 use Illuminate\Support\Facades\Log;
 use App\Lib\QrCodeHelper;
+use Shopify\Clients\Graphql;
 
 class QrCodeController extends Controller
 {
     private $qrCodeHelper;
 
+    private const SHOP_DATA_QUERY = <<<'QUERY'
+    query shopData($first: Int!) {
+        shop {
+            url
+        }
+        codeDiscountNodes(first: $first) {
+            edges {
+                node {
+                    id
+                    codeDiscount {
+                    ... on DiscountCodeBasic {
+                        codes(first: 1) {
+                        edges {
+                            node {
+                            code
+                            }
+                        }
+                        }
+                    }
+                    ... on DiscountCodeBxgy {
+                        codes(first: 1) {
+                            edges {
+                                node {
+                                code
+                                }
+                            }
+                        }
+                    }
+                    ... on DiscountCodeFreeShipping {
+                        codes(first: 1) {
+                            edges {
+                                node {
+                                code
+                                }
+                            }
+                        }
+                    }
+                    }
+                }
+            }
+        }
+    }
+    QUERY;
+
     public function __construct(QrCodeHelper $qrCodeHelper)
     {
        $this->qrCodeHelper = $qrCodeHelper;
     }

...

// 以下のメソッドを追加

    /**
     * ディスカウントをshopifyから取得
     *
     * @param  Request  $request
     * @return \Illuminate\Http\JsonResponse
     */
    public function getDisCounts(Request $request)
    {
        $session = $request->get('shopifySession');
        $client = new Graphql($session->getShop(), $session->getAccessToken());

        $shopData = $client->query([
            'query' => self::SHOP_DATA_QUERY,
            'variables' => ['first' => 25],
        ])->getDecodedBody();

        return response()->json($shopData['data']);
    }
```
上記まででQRコードのCRUD処理は終わりです！
後は実際にQRコードを生成し、QRコードからショップ画面に遷移するか確認するのみです。

# QRコードの生成
## QRコード読み取りのルートを許可する
ここまでのバックエンドとの連携は/apiで始まるルートしか許可されていません。
viteのconfigでプロキシのルールに/qrcodes ルートを許可するよう設定みたいです。
```diff js:web/frontend/vite.config.js
export default defineConfig({
  root: dirname(fileURLToPath(import.meta.url)),
  plugins: [react()],
  define: {
    "process.env.SHOPIFY_API_KEY": JSON.stringify(process.env.SHOPIFY_API_KEY),
  },
  resolve: {
    preserveSymlinks: true,
  },
  server: {
    host: "localhost",
    port: process.env.FRONTEND_PORT,
    hmr: hmrConfig,
    proxy: {
      "^/(\\?.*)?$": proxyOptions,
      "^/api(/|(\\?.*)?$)": proxyOptions,
+     "^/qrcodes/[0-9]+/image(\\?.*)?$": proxyOptions,
+     "^/qrcodes/[0-9]+/scan(\\?.*)?$": proxyOptions,
    },
  },
});
```

## routeの追加
QrCodePublicControllerや各メソッドはこれから実装します。
```php:web/routes/web.php
use App\Http\Controllers\QrCodePublicController;

...

Route::get('/qrcodes/{id}/image', [QrCodePublicController::class, 'applyQrCodePublic']);
Route::get('/qrcodes/{id}/scan', [QrCodePublicController::class, 'scan']);
```

## QRコードライブラリの追加
チュートリアルではqrcodeというライブラリを利用してましたが、phpでやりたいので、php用のライブラリを探しました。
simple-qrcodeというライブラリが一番使いやすそうだったので、導入します。
https://github.com/SimpleSoftwareIO/simple-qrcode
※PHPでImagick拡張しないとエラーになると思います。
ここに書いてあるかな？
https://www.php.net/manual/ja/book.imagick.php

以下のコマンドで早速インストールします。
```
$ cd web
$ composer require simplesoftwareio/simple-qrcode
```
## QRコード生成
QrCodePublicControllerを作成します。
残りQRコードの生成とスキャンされた時の処理がありますが、一気に実装しちゃいます。
```php:web/app/Http/Controllers/QrCodePublicController.php
<?php

namespace App\Http\Controllers;

use App\Lib\QrCodeHelper;
use Shopify\Context;
use SimpleSoftwareIO\QrCode\Facades\QrCode;
use Illuminate\Support\Facades\Redirect;

class QrCodePublicController extends Controller
{
    private $qrCodeHelper;
    private const DEFAULT_PURCHASE_QUANTITY = 1;

    public function __construct(QrCodeHelper $qrCodeHelper)
    {
        $this->qrCodeHelper = $qrCodeHelper;
    }

    public function applyQrCodePublic($id)
    {
        $qrCode = $this->qrCodeHelper->getQrCodeOr404($id, false);
        if ($qrCode) {
            $destinationUrl = $this->__generateQrcodeDestinationUrl($qrCode);
            $image = QrCode::size(150)
                ->gradient(100, 100, 200, 20, 20, 100, 'vertical')
                ->format('png')
                ->generate($destinationUrl);
            $headers = [
                'Content-Type' => 'image/png',
                'Content-Disposition' => 'inline; filename="qr_code_' . $qrCode->id . '.png"',
            ];

            return response($image, 200, $headers);
        }
    }

    public function scan($id)
    {
        $qrCode = $this->qrCodeHelper->getQrCodeOr404($id, false);
        $qrCode->where('id', $id)
            ->update([
                'scans' => $qrCode->scans + 1
            ]);
        $url = $qrCode->shopDomain;
        switch ($qrCode->destination) {
            case "product":
                return $this->__goToProductView($url, $qrCode);
            case "checkout":
                return $this->__goToProductCheckout($url, $qrCode);
            default:
                return 'Unrecognized destination.qrCode.destination';
        }
    }

    private function __generateQrcodeDestinationUrl($qrCode)
    {
        return Context::$HOST_SCHEME.'://'.Context::$HOST_NAME.'/qrcodes/'.$qrCode['id'].'/scan';
    }

    private function __goToProductView($url, $qrCode)
    {
        $discountCode = $qrCode['discountCode'];
        $productHandle = $qrCode['handle'];

        $urlString = $this->__productViewURL($url, $productHandle, $discountCode);

        return Redirect::to($urlString);
    }

    private function __goToProductCheckout($url, $qrCode)
    {
        $discountCode = $qrCode['discountCode'];
        $variantId = $qrCode['variantId'];
        $quantity = self::DEFAULT_PURCHASE_QUANTITY;

        $urlString = $this->__productCheckoutURL($url, $variantId, $quantity, $discountCode);

        return Redirect::to($urlString);
    }


    private function __productViewURL($url, $productHandle, $discountCode = null)
    {
        $productPath = '/products/'.urlencode($productHandle);

        if ($discountCode) {
            $redirectUrl = $url.'/discount/'.$discountCode.'?redirect='.$productPath;
        } else {
            $redirectUrl = $url.$productPath;
        }

        return $redirectUrl;
    }

    private function __productCheckoutURL ($url, $variantId, $quantity = 1, $discountCode = null)
    {
        $id = preg_replace('/gid:\/\/shopify\/ProductVariant\/([0-9]+)/', '$1', $variantId);
        $cartPath = "/cart/{$id}:{$quantity}";

        $redirectUrl = $url.$cartPath;

        if ($discountCode) {
            $redirectUrl .= '?discount=' . urlencode($discountCode);
        }

        return $redirectUrl;
    }
}
```

以上で動作確認できれば完成です！
# 色ランダムに変えてみる
QRコードの色を自由に変えられるようなので、毎回ランダムに変更されるようにしてみます。
```diff php:web/app/Http/Controllers/QrCodePublicController.php
 public function applyQrCodePublic($id)
    {
        $qrCode = $this->qrCodeHelper->getQrCodeOr404($id, false);
        if ($qrCode) {
            $destinationUrl = $this->__generateQrcodeDestinationUrl($qrCode);
            $image = QrCode::size(150)
-               ->gradient(100, 100, 200, 20, 20, 100, 'vertical')
+               ->gradient(rand(0, 255), rand(0, 255), rand(0, 255), rand(0, 255), rand(0, 255), rand(0, 255), 'vertical')
                ->format('png')
                ->generate($destinationUrl);
            $headers = [
                'Content-Type' => 'image/png',
                'Content-Disposition' => 'inline; filename="qr_code_' . $qrCode->id . '.png"',
            ];

            return response($image, 200, $headers);
        }
    }
```

草

# やってみた感想
ドキュメントは英語だし、量も多いし、初見ではなかなかキツかったです。
Shopifyは凄いと思いましたが、まだまだわからないこと多いので、これからじっくりやっていきたいです。
あと、アップデート早すぎて無理。
最後までお読みいただき、ありがとうございました🙇‍♂️