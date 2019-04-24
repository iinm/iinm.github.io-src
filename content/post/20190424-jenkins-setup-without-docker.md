---
title: "WIP: 非docker環境下でのJenkinsのセットアップの自動化"
date: 2019-04-24T20:51:21+09:00
draft: false
---

# 背景

とある環境で、大体のものはAnsibleで構成管理されているが、Jenkinsに関してはインストールまではAnsible、そこから先の設定は手動、という状況に出会った。なにか問題が発生した際に手動で再設定するのも面倒だなと思い、セットアップを自動化できないか調べた。
最近は必要なプラグインを同封したコンテナを作ることが多いみたいだが、コンテナ前提となるといろいろと変えなければならないことが多いので [jenkinsci/docker](https://github.com/jenkinsci/docker) は参考にはするけど使わない。

# 自動化のスコープ

- 初回セットアップ
- 管理者ユーザの作成
- プラグインのインストール
- ユーザの追加

↑↑ 今回はここまで。 ↑↑

- プラグインの設定
- ユーザの権限設定
- credentialの追加
- Jobの追加

# どのように実現するか？

## 初回セットアップのスキップ、管理者ユーザの作成

通常はセットアップ画面が表示され、プラグインのインストール、管理者ユーザの作成、JenkinsのURLの設定を画面に従って進める必要がある。

セットアップをスキップするにはプロパティをセットするだけ。
```sh
java -Djenkins.install.runSetupWizard=false -jar jenkins.war
```

ただし、その副作用としてSecurity設定がOFFの誰でも触れるJenkinsが起動してしまうため、初回起動時に実行するスクリプトを用意する。
何らかのeventに対し任意のGroovyスクリプトを実行することが可能らしい。
[Groovy Hook Script](https://wiki.jenkins.io/display/JENKINS/Groovy+Hook+Script)

例えば、初回起動時に実行するスクリプトは `$JENKINS_HOME/init.groovy` という名前で用意する。
```groovy
import jenkins.model.*
import hudson.security.*
import hudson.security.csrf.*

def env = System.getenv()
def jenkins = Jenkins.getInstance()

// set url
urlConfig = JenkinsLocationConfiguration.get()
urlConfig.setUrl(env.JENKINS_URL)
urlConfig.save()

// create admin user
if (!(jenkins.getSecurityRealm() instanceof HudsonPrivateSecurityRealm)) {
    jenkins.setSecurityRealm(new HudsonPrivateSecurityRealm(false))
}
def user = jenkins.getSecurityRealm().createAccount(env.JENKINS_USER, env.JENKINS_PASSWORD)
user.save()

// configure security
def strategy = new FullControlOnceLoggedInAuthorizationStrategy()
strategy.setAllowAnonymousRead(false)
jenkins.setAuthorizationStrategy(strategy)

// enable csrf protection
if (jenkins.getCrumbIssuer() == null) {
    jenkins.setCrumbIssuer(new DefaultCrumbIssuer(true))
}

// disable cli
jenkins.getDescriptor("jenkins.CLI").get().setEnabled(false)

jenkins.save()
```

## プラグインのインストール

Jenkins CLIでインストールできる。
```sh
for plugin in `cat plugins.txt`; do
  java -jar jenkins-cli.jar -s $(jenkins_url) -auth $(jenkins_cli_auth) install-plugin $plugin -deploy
done
```

[Jenkinsを起動する前にプラグインをインストールできるスクリプト](https://github.com/jenkinsci/docker/blob/master/install-plugins.sh)もあるのだが、若干修正が必要だったのと、やろうとしていることに対して複雑すぎると感じたので使わなかった。

## ユーザの追加

これもJenkins CLIでGroovy scriptを実行することで可能
```sh
echo 'jenkins.model.Jenkins.instance.securityRealm.createAccount("$(username)", "$(password)")' \
  | java -jar jenkins-cli.jar -s $(jenkins_url) -auth $(jenkins_cli_auth) -noKeyAuth groovy = –
```

# まとめ & メモ

[Makefile](https://github.com/iinm/jenkins-template)

```sh
make jenkins_user=admin jenkins_password=$password run  # 起動 & 管理者作成
make jenkins_plugin_file=plugins.txt install-plugins    # pluginのインストールする
make username=jenkins password=$password add-user       # ユーザの追加
```
