# Yellowfinのlog4j2設定の変更

log4j2.xml を変更して、hostname別のディレクトリにログを出力するように変更する。

## log4j2.xml ファイルの抽出

yellowfinアプリケーションのWEB-INFディレクトリにlog4j2の設定ファイルは保存されている。
`/opt/yellowfin/appserver/webapps/ROOT/WEB-INF`

all-in-one バージョンをローカルで起動して、log4j2.xml を抽出。
```
docker run -d --rm -n yellowfin yellowfinbi/yellowfin-all-in-one
docker cp yellowfin:/opt/yellowfin/appserver/webapps/ROOT/WEB-INF/log4j2.xml log4j2.xml
```

## log4j2.xml ファイルの編集

ローカルに保存されたlog4j2.xmlを編集

`<Properties>`　セクションにログ出力ディレクトリを指定するためのプロパティを設定し、ログのAppenderのログ出力ディレクトリに指定する。
```
<Properties>
  <Property name="hostname">${env:HOSTNAME}</Property>
</Properties>

＜＜＜中略＞＞＞

<Appenders>
		<RollingFile name="applog" fileName="${logDir}/${hostname}/yellowfin.log" filePattern="${logDir}/${hostname}/yellowfin.log.%i">
			<PatternLayout pattern="${appenderPatternLayout}" />			
			<Policies>
				<SizeBasedTriggeringPolicy size="${maxFileSize}" />
			</Policies>
			<DefaultRolloverStrategy fileIndex="min" max="${maxFiles}"/>
		</RollingFile>
</Appenders>
```

##　コンテナイメージの作成

編集後のlog4j2.xmlを含むコンテナイメージの作成は、既存コンテナイメージを更新して保存。
※ただし、この場合ランタイムに生成されるファイルも保存してしまうので、一時的な利用。


### 方法1: docker commit

編集したファイルをコンテナイメージにコピーして、docker commit コマンドで保存。

スキーム
```
docker commit 実行中のコンテナ 保存するコンテナイメージ名[:tag]
```

```
docker cp log4j2.xml yellowfin:/opt/yellowfin/appserver/webapps/ROOT/WEB-INF/log4j2.xml
docker commit yellowfin myyellowfin
```

改めて、保存したコンテナイメージで実行するとログ出力ディレクトリが変更されていることが確認きる。


### 方法2: docker build

Azure Container Registryを使ったビルド
```
export ACR_NAME=myacr
az acr build -r $ACR_NAME -t myyellowfin --platform linux .
```



## 参考
https://logging.apache.org/log4j/2.x/manual/configuration.html#Properties
