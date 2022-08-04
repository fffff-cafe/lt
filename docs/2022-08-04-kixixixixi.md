# Github Actions
2022-08-04 @kixixixixi

---

## Next.jsをSSGしてS3にデプロイ
mainプッシュごとに以下の処理を行いデプロイ
- SSG ビルド
- S3へのアップロード
- Cloudfrontのキャッシュ削除

next.jsでSSGする処理はpackage.jsonに
```json
{
  "scripts": {
    "build": "NODE_ENV=production next build && next export"
  },
  ...
}
```

```yaml
name: Deploy to production

on:
  push:
    branches:
      - main

jobs:
  build:
    name: Deploy to production S3 and invalidate cloundfront cache
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3
        # code取得

      - uses: actions/setup-node@v3
        with:
          node-version: 16
          cache: npm
        # nodeのキャッシュ設定

      - name: Install packages
        run: yarn
        # yarn もしくは npm ci で依存関係の取得が短くできるはず

      - name: Build
        run: yarn build
        env:
          ...
          # ビルドに付与するenv設定

      - name: Deploy to S3
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: aws s3 sync --region ap-northeast-1 out/ ${{ secrets.S3_URL }}
        # outに出力されたhtmlなどをs3にアップロード

      - name: Create cache invalidation
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_REGION: ap-northeast-1
        run: aws cloudfront create-invalidation --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} --paths "/*"
        # デプロイ後にもcloudfrontでキャッシュがきいてしまうので明示的にキャッシュ削除
```

---

## Next.jsでSSGしてGithub pagesにデプロイ
Github pagesの設定をSettingsから行うと勝手にデプロイしてくれる

```yaml
name: deploy

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-18.04
    steps:
      - uses: actions/checkout@v2

      - name: setup node
        uses: actions/setup-node@v3
        with:
          node-version: "16.x"

      - name: Cache dependencies
        uses: actions/cache@v1
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-
        # npm installのキャッシュをきかせたいけどうまくいっているのか謎
        
      - name: Install
        run: npm ci

      - name: Build
        run: npm run build
        # "build": "NODE_ENV=production next build && next export"

      - name: Add nojekyll
        run: touch ./out/.nojekyll
        # Github pagesでHTML表示させるための設定

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./out
          cname: ${{ secrets.DOMAIN }}
        # github pagesの設定をする。このactionのあとにgithub pagesのactionが動く。cnameの設定がないのでsettingsで定義したドメイン設定がいつも外れるので注意
```

---

## DockerビルドしてECSにデプロイ
前もってECSの環境をコンソールもしくはCloudformationなどで用意しておく（Fargate）
タグの発行ごとに以下のデプロイを行う
- Dockerビルド
- ECRへのプッシュ
- ECSタスク定義の更新
- サービスの更新

```yaml
name: Production deploy

on:
  push:
    tags:
      - '*'

jobs:
  production-deploy:

    runs-on: ubuntu-latest
    timeout-minutes: 300

    steps:
    - name: Checkout
      uses: actions/checkout@v2

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v1
      # Docker向けキャッシュ

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ap-northeast-1
      # AWS認証情報の設定

    - name: Login to ecr
      uses: docker/login-action@v1
      with:
        registry: ${{ secrets.ECR_ENDPOINT }}/${{ secrets.REPOSITORY_NAME }}
      # ECRへのログイン

    - name: Extract Git Tag
      run: echo "GIT_TAG=${GITHUB_REF/refs\/tags\//}" >> $GITHUB_ENV
      # イメージのタグを用意

    - name: Build and push
      uses: docker/build-push-action@v2
      with:
        context: .
        push: true
        tags: |
          ${{ secrets.ECR_ENDPOINT }}/${{ secrets.REPOSITORY_NAME }}:latest
          ${{ secrets.ECR_ENDPOINT }}/${{ secrets.REPOSITORY_NAME }}:${{ env.GIT_TAG }}
        cache-from: ${{ secrets.ECR_ENDPOINT }}/${{ secrets.REPOSITORY_NAME }}
      # DockerビルドとECRへのpush、キャッシュがうまくきいているのか...?
      
    - name: Download task definition
      run: >-
        aws ecs describe-task-definition --task-definition ai-api-prod-task-def --query taskDefinition
        | jq 'del (.taskDefinitionArn, .revision, .status, .requiresAttributes, .compatibilities, .registeredAt, .registeredBy)' > task-definition.json
      # Task定義を取得し不要な項目を削る
      
    - name: Fill in the new image ID in the Amazon ECS task definition
      uses: aws-actions/amazon-ecs-render-task-definition@v1
      id: task-def
      with:
        task-definition: task-definition.json
        container-name: ${{ secrets.CONTAINER_NAME }}
        image: ${{ secrets.ECR_ENDPOINT }}/${{ secrets.REPOSITORY_NAME }}
      # Task定義のイメージ名を書き換え

    - name: Deploy Amazon ECS task definition
      uses: aws-actions/amazon-ecs-deploy-task-definition@v1
      with:
        task-definition: ${{ steps.task-def.outputs.task-definition }}
        service: ${{ secrets.ECS_SERVICE_NAME }}
        cluster: ${{ secrets.ECS_CLUSTER_NAME }}
        wait-for-service-stability: true
      # Task定義を反映。サービスが更新されコンテナが入れ替わると終了する
```

---

## 感想

- Slack等への通知をどうにかしたい
```yaml
      - name: Notify
        uses: 8398a7/action-slack@v3.8.0
        with:
          status: ${{ job.status }}
          fields: repo,message,commit,author,action,job,took,eventName,ref,workflow # 追加
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
        if: always()
```
- cron等がonで使えるので定時バッジ等動かせそう
```yaml
on:
  schedule:
    - cron:  '30 5,17 * * *'
```
