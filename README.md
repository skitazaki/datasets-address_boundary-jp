Datasets MLIT Address Boundary
==============================

国土交通省の[国土数値情報・行政区域データ](http://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-N03.html)

## 環境の準備

データベースを作成し、PostGIS 拡張を有効にする。

    $ createdb -U postgres geo

    $ cat << EOF | psql -U postgres -d geo
    CREATE EXTENSION postgis;
    CREATE EXTENSION postgis_topology;
    CREATE EXTENSION fuzzystrmatch;
    CREATE EXTENSION postgis_tiger_geocoder;
    SELECT postgis_full_version();
    EOF

最終的にデータを入れるテーブルを用意する。

    $ psql -U postgres -d geo -f address_boundary.sql


## データのインポート

ダウンロードした ZIP ファイルを展開する。

    $ for f in N03-*.zip
    do
        unzip -d ${f%.zip} $f
    done

Shape 形式のファイルを PostGIS にインポートするための SQL ファイルに変換する。

    $ for f in N03-*/*.shp
    do
        t=`basename $f`
        sql=${t%.shp}.sql
        shp2pgsql -W "Shift_JIS" $f > $sql
    done

出来上がった SQL ファイルを先ほど作成したデータベースに対して実行する。

    $ for f in N03-*.sql
    do
        psql -U postgres -d geo -f $f
    done

## データの確認と修正

住所コードで集約したときに名称が不一致になるレコードを抽出する。

    $ for i in {01..47}
    do
        psql -U postgres -d geo -t -f checkdup.sql -v table="n03-13_${i}_130401"
    done | grep "|"

名称が不一致のレコードを修正する。

    $ psql -U postgres -d geo -f patch.sql

## データの変換

住所コードをキーにして単一のテーブルにデータを集約する。

    $ for i in {01..47}
    do
        psql -U postgres -d geo -f transform.sql -v table="n03-13_${i}_130401"
    done

テーブルの件数を確認する。

    $ echo "SELECT count(*) FROM address_boundary;" | psql -U postgres -d geo

## データの出力

ダンプファイルを作成する。

    $ pg_dump -F c -x -U postgres -d geo -t address_boundary -f address_boundary.backup

S3に保存する。

    $ S3_BUCKET=s.kitazaki.name
    $ S3_PATH=datasets/mlit
    $ fname=address_boundary.backup
    $ aws s3 cp $fname s3://$S3_BUCKET/$S3_PATH/$fname


## データの出典

* 「国土数値情報（行政区域データ）　国土交通省」
* 平成２５年度データ

