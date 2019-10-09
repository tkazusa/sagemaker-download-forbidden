# SagemakerでJupyter Notebookのダウンロードを禁止
`jupyter_notebook_config.py` の `c.ContentsManager.files_handler_class` がファイルのハンドリングを制御しているため、
`FilesHandler`を下記のように書き換えることで、ダウンロードに対して `403:Forbidden error` を返すようにできる。

```python
from tornado import web
from notebook.base.handlers import IPythonHandler

class ForbidFilesHandler(IPythonHandler):
    @web.authenticated
    def head(self, path):
        self.log.info("HEAD: File download forbidden.")
        raise web.HTTPError(403)

    @web.authenticated
    def get(self, path, include_body=True):
        self.log.info("GET: File download forbidden.")
        raise web.HTTPError(403)
```

Amazon SageMaker は`/home/ec2-user/SageMaker` 以外の変更を保存しないので、`/home/ec2-user/.jupyter/handlers.py` への変更は ライフサイクル設定を使う必要がある。ノートブックを開始する際のスクリプトを下記とする。


```python
# Creating ForbidFilesHandler class, overriding the default files_handler_class
cat <<END >/home/ec2-user/.jupyter/handlers.py
from tornado import web
from notebook.base.handlers import IPythonHandler

class ForbidFilesHandler(IPythonHandler):
    @web.authenticated
    def head(self, path):
        self.log.info("HEAD: File download forbidden.")
        raise web.HTTPError(403)

    @web.authenticated
    def get(self, path, include_body=True):
        self.log.info("GET: File download forbidden.")
        raise web.HTTPError(403)

END

# Updating the files_handler_class
cat <<END >>/home/ec2-user/.jupyter/jupyter_notebook_config.py
import os, sys
sys.path.append('/home/ec2-user/.jupyter/')
import handlers
c.ContentsManager.files_handler_class = 'handlers.ForbidFilesHandler'
c.ContentsManager.files_handler_params = {}

END

# Reboot the Jupyterhub notebook
reboot
```
さらに強固に行うのであれば、Amazon WorkSpaces か AppStream 2.0をご利用いただける。

特にAppStream 2.0
- Private EndpointからStreamingできるので、社内にPrivate DXが引かれているのであれば、ネットワークも閉域からのアクセスが可能になる
- ユーザー認証は内部のUser Poolを使うか、数が多い場合はSAML連携が可能
- ADで動かすのであれば、外部のSSOと連携するか、ADFSで稼働させる

## 参考
- [Disable Download Button on the SageMaker Jupyter Notebook](https://ujjwalbhardwaj.me/post/disable-download-button-on-the-sagemaker-jupyter-notebook)
