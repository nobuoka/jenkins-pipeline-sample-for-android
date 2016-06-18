// Jenkinsfile に書かれた Groovy のスクリプトは Script Security Plugin によりセキュリティチェックされる。
// 許可されていないメソッドを使用すると `RejectedAccessException` が発生するので、その場合は
// Jenkins の管理者権限でメソッドを許可すると良い。
// See : http://qiita.com/nobuoka/items/0f8cc736d8e6439aa4ac

/** ビルドに用いる定数変数を置くクラス。 */
class C {
    // トークンの置き場所は運用方法やポリシー等に応じて変更すること。
    static final String GHE_TOKEN = 'YOUR_TOKEN'
    static final String GHE_API = 'https://your.ghe.domain/api/v3'
    static final String GHE_REPO = 'your_team/repository_name'

    // Web hook の URL の置き場所は運用方法やポリシー等に応じて変更すること。
    static final String SLACK_WEBHOOK_URL = 'https://hooks.slack.com/services/xxxxxx'
}

node {
    withSlackNotifier() {
        def currentPath = pwd()
        // こういう場所に置いていいものかどうか……？
        env.ANDROID_HOME = currentPath + '/../.android-sdk'

        stage 'Environment'
        sh 'java -version'

        stage 'Check Android SDK'
        if (fileExists(env.ANDROID_HOME)) {
            echo 'Android SDK already exists'
        } else {
            stage 'Setup Android SDK'
            // 富豪的なのでもうちょっといい方法を考えたい。
            sh 'curl --fail --output android-sdk.tgz http://dl.google.com/android/android-sdk_r24.4.1-linux.tgz'
            sh 'tar -xvf android-sdk.tgz'
            sh 'mv android-sdk-linux "$ANDROID_HOME"'
        }

        // Mark the code checkout 'stage'....
        stage 'Checkout'
        // Checkout code from repository
        checkout scm

        // コミットハッシュを取得
        sh 'git rev-parse --verify HEAD > jenkins_pipeline_output.temp'
        def commitHash = readFile('jenkins_pipeline_output.temp').split("\r?\n")[0]
        env.GIT_COMMIT = commitHash

        stage 'Assemble'
        withGheStatusSender('Assemble', env.GIT_COMMIT, 'Building') {
            sh './gradlew assemble'
        }

        stage 'Lint'
        withGheStatusSender('Lint', env.GIT_COMMIT, 'Checking') {
            // lint タスクを実行した場合、全 variant に対して lint が実行されるが、出力は 1 つしかないないぽい。
            // とりあえずは各 variant を明示して lint することで対応する。
            def pfs = ['']
            def bts = ['production', 'debug']
            List<String> variants = []
            pfs.each { pf -> bts.each{ bt -> variants.add(pf + (pf.isEmpty() ? bt : bt.capitalize())) } }

            // List#collect メソッドを使いたいが、groovy-cps ライブラリのバグで Pipeline 上でうまく動かない。
            // See : https://issues.jenkins-ci.org/browse/JENKINS-26481
            List<GString> gradleTasks = []
            List<GString> outputFiles = []
            variants.each {
                gradleTasks.add(":app:lint${it.capitalize()}")
                outputFiles.add("lint-results-${it}.html")
            }

            // テスト失敗時にも結果を保存するように try-catch する。
            Throwable error = null
            try { sh "./gradlew --stacktrace ${gradleTasks.join(' ')}" } catch (e) { error = e }
            try {
                publishHTML([
                        target: [
                                reportName: 'Android Lint Report',
                                reportDir: 'app/build/outputs/',
                                reportFiles: outputFiles.join(','),
                        ],
                        // Lint でエラーが発生した場合はリポートファイルがないことを許容する。
                        allowMissing: error != null,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                ])
            } catch (e) { if (error == null) error = e }
            if (error != null) throw error
        }

        stage 'Local Unit Test'
        withGheStatusSender('Local Unit Test', env.GIT_COMMIT, 'Testing') {
            runTestAndArchiveResult(':app:test', 'app/build/test-results', '*/TEST-*.xml')
        }

        // Emulator にバージョンアップで以下のものが使えなくなったので一旦コメントアウト。
        // 社内では shell スクリプトで AVD の起動や終了をするようにした。
        /*
        stage 'Instrumented Test'
        try {
            sh './gradlew :avd:startAvd'
            sh './gradlew connectedAndroidTest'
        } finally {
            sh './gradlew :avd:killAvd'
        }
        */
    }
}

/** GHE にステータスを通知する。 */
void postGheStatus(Map<String, String> params) {
    // 送信する JSON をファイルに書き出しておく。
    String jsonFileName = 'jenkins_pipeline_input_json.temp'
    String jsonContent = groovy.json.JsonOutput.toJson([
            state: params['state'],
            target_url: env.BUILD_URL,
            description: params['description'],
            context: params['context'],
    ])

    writeFile file: jsonFileName, text: jsonContent
    // 実際に送信する際にはここのコメントアウトを外す。
    /*
    sh 'curl --insecure -H "Authorization: token ' + C.GHE_TOKEN + '" ' +
            '"' + C.GHE_API + '/repos/' + C.GHE_REPO + '/statuses/' + params['commitHash'] + '" ' +
            '-X POST ' +
            '-d @' + jsonFileName
    */
}

/**
 * GHE へのステータス通知を行ってタスクの実行を行う。
 * タスク実行前に pending 状態を通知し、タスク完了後に、タスクの結果に応じて成功か失敗の状態を通知する。
 */
void withGheStatusSender(String context, String commitHash, String firstDescription, Closure task) {
    postGheStatus(context: context, commitHash: commitHash, state: 'pending', description: firstDescription)
    try {
        task()
        postGheStatus(context: context, commitHash: commitHash, state: 'success', description: 'Success')
    } catch (e) {
        postGheStatus(context: context, commitHash: commitHash, state: 'failure', description: 'Failure')
        throw e
    }
}

/** Slack に投稿する。 */
void postSlack(String text, boolean useNgJenkinsIcon) {
    String slackUrl = C.SLACK_WEBHOOK_URL
    String iconImageUrl = useNgJenkinsIcon ?
            '成功時の Jenkins アイコン' :
            '失敗時の Jenkins アイコン'
    String payload = groovy.json.JsonOutput.toJson([
            text: text,
            icon_url: iconImageUrl,
    ])
    // 実際に送信する際にはここのコメントアウトを外す。
    //sh "curl -X POST --data-urlencode \'payload=${payload}\' ${slackUrl}"
}

/** タスクを実行し、実行後に成功か失敗かを Slack に投稿する。 */
void withSlackNotifier(Closure task) {
    echo "branch name : ${env.BRANCH_NAME}"
    try {
        task()
        postSlack("Job for branch `${env.BRANCH_NAME}` succeeded! (<${env.BUILD_URL}|Open>)", false)
    } catch (e) {
        postSlack("Job for branch `${env.BRANCH_NAME}` failed! (<${env.BUILD_URL}|Open>)", true)
        throw e
    }
}

void runTestAndArchiveResult(String gradleTestTask, String resultsDir, String resultFilesPattern) {
    // `test` タスクの入力と出力の両方とも更新がなければ `test` タスクがスキップされる (UP-TO-DATE) ので、
    // スキップされないように出力ディレクトリを消しておく。
    sh "rm -rf ${resultsDir}"

    // テスト失敗時にも結果を保存するように try-catch する。
    Throwable error = null
    try { sh "./gradlew --stacktrace ${gradleTestTask}" } catch (e) { error = e }
    try {
        step $class: 'JUnitResultArchiver', testResults: "${resultsDir}/${resultFilesPattern}"
    } catch (e) { if (error == null) error = e }
    if (error != null) throw error
}
