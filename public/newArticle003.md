---
title: newArticle003
tags:
  - ""
private: false
updated_at: ""
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# この記事で書くこと

## migration の設定

> on delete cascade

- delete cascade は設定しておいた方が開発しやすい

```php:database/migrations/2024_12_15_212048_skill_and_related_tables.php
...
    public function up(): void
    {
        Schema::create('skills', function (Blueprint $table) {
            $table->id();
            $table->unsignedBigInteger('user_id'); //外部キー
            $table->string('name', 20);
            $table->timestamps();

            $table->foreign('user_id')->references('id')->on('users')->onDelete('cascade');
        });
...
```

##

# メリット・デメリット

## on delete cascade

- メリット：開発時にデータの削除も簡単
- デメリット：本番でミスると全部消える
