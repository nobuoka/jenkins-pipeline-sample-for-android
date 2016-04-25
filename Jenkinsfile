node {
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

    stage 'Build'
    sh './gradlew -PsetupAndroidSdk assemble'

    stage 'Lint'
    sh './gradlew lint'

    stage 'Local Unit Test'
    sh './gradlew test'

    stage 'Instrumented Test'
    try {
        sh './gradlew :avd:startAvd'
        sh './gradlew connectedAndroidTest'
    } finally {
        sh './gradlew :avd:killAvd'
    }
}
