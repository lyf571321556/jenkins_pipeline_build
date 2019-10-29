import groovy.transform.Field

def git_creds = 'company_svc_c_jenkins_ssh_key'
def artifactory_creds = 'company_artifactory_jenkins_creds'
def mp_api_key = '17298f767ae7a866e0363776e5ZZZZZZ'
def mp_api_secret = '58eec9da7fd8fe6f30bc9XXXXXX'
@Field def sendTo = 'itai.ganot@company.com'
@Field def channel = '#slack-test'


def check_test_results(String path) {
    try {
        step([
            $class: 'XUnitBuilder',
            testTimeMargin: '3000',
            thresholdMode: 1,
            thresholds: [
                [$class: 'FailedThreshold', failureNewThreshold: '0', failureThreshold: '0', unstableNewThreshold: '', unstableThreshold: ''],
                [$class: 'SkippedThreshold', failureNewThreshold: '', failureThreshold: '', unstableNewThreshold: '', unstableThreshold: '']
            ],
            tools: [
                [$class: 'JUnitType', deleteOutputFiles: false, failIfNotNew: false, pattern: path, skipNoTestFiles: false, stopProcessingIfError: true]
            ]
        ])
    }
    catch(error) {
        slackSend channel: channel, color: 'danger', teamDomain: null, token: null,
            message: "${ulink}: *Failed to build ${env.JOB_NAME}*! :x: ${jlink}(<!here|here>)"
    }
}


pipeline {
    agent{
    label "agent"
    }
    Map started_by = utils.get_started_by()
    String ulink = "<@${started_by['userId']}>"
    String jlink = "(<${env.BUILD_URL}|Open>)"

    stage('Git clean, Checkout SCM'){
        sh '( git reset --hard; git clean -fxd --exclude=".gradle" ; git tag -d $(git tag) ) &>/dev/null || true'
        checkout scm
    }

    def cwd = pwd()

    stage('Environement preparation') {
        // Build parameters
        NDK_VER="r12b"
        SDK_VER="r24.4.1"
        GRADLE_USER_HOME="${cwd}/.gradle"
        NDK_DIR="${GRADLE_USER_HOME}/android-ndk-${NDK_VER}"
        ANDROID_HOME="${GRADLE_USER_HOME}/android-sdk-linux"
        SDK_TOOLS="${ANDROID_HOME}/tools"
        AAPT="${ANDROID_HOME}/build-tools/23.0.3"

        withCredentials([ // Use Jenkins credentials ID of artifactory
            [$class: 'UsernamePasswordMultiBinding', credentialsId: artifactory_creds, usernameVariable: 'A_USER', passwordVariable: 'A_PASS'],
        ]){
            sshagent (credentials: [git_creds]){
                // Updates submodules recursively
                sh """ git submodule update --init --recursive """
            }
            sh """
                export PATH="\$PATH:${GRADLE_USER_HOME}:${NDK_DIR}:${ANDROID_HOME}:${SDK_TOOLS}:${AAPT}"

                # Checking that Android SDK and NDK are installed
                mkdir -p ${GRADLE_USER_HOME}
                if [ ! -f "${GRADLE_USER_HOME}/android-sdk-${SDK_VER}-linux.tgz" ]; then
                    curl -o "${GRADLE_USER_HOME}/android-sdk-${SDK_VER}-linux.tgz" https://dl.google.com/android/android-sdk_${SDK_VER}-linux.tgz
                fi

                if [ ! -f "${GRADLE_USER_HOME}/android-ndk-${NDK_VER}-linux-x86_64.zip" ]; then # Checks if the sdk tarball exists on system
                    curl -o "${GRADLE_USER_HOME}/android-ndk-${NDK_VER}-linux-x86_64.zip" https://dl.google.com/android/repository/android-ndk-${NDK_VER}-linux-x86_64.zip
                fi

                if [ ! -d "${GRADLE_USER_HOME}/android-ndk-${NDK_VER}" ]; then
                    echo "android-ndk DOESNT EXIST!"
                    cd "${GRADLE_USER_HOME}" && unzip -oqa "android-ndk-${NDK_VER}-linux-x86_64.zip"
                fi

                if [ ! -d "${ANDROID_HOME}" ]; then
                    echo "Android-sdk DOESNT EXIST!"
                    tar -xzf "${GRADLE_USER_HOME}/android-sdk-${SDK_VER}-linux.tgz" -C ${GRADLE_USER_HOME}
                    mkdir "${GRADLE_USER_HOME}/android-sdk-linux/extras"
                fi

                #Configuring path to ndk and sdk dir - as required by Gradle
                printf "ndk.dir=$NDK_DIR\nsdk.dir=$ANDROID_HOME" > "$cwd/local.properties"
                cp "$cwd/local.properties" "$cwd/Product-AndroidSDK"
                cp "$cwd/local.properties" "$cwd/Product-ServicesSDK"
                cp "$cwd/local.properties" "$cwd/Product-CoreSDK"

                # Creates licenses folder which will include the signed licenses for google sdk packages
                mkdir "${ANDROID_HOME}/licenses"
                echo -e "\n8933bad161af4178b1185d1a37fbf41ea5269c55" > "${ANDROID_HOME}/licenses/android-sdk-license"
                echo -e "\n84831b9409646a918e30573bab4c9c91346d8abd" > "${ANDROID_HOME}/licenses/android-sdk-preview-license"

                # Mandatory manual downloads of required SDK tools
                echo "y" | android update sdk -u -a -t platform-tools                    # Android SDK Platform-tools, revision 25
                echo "y" | android update sdk -u -a -t tools                             # Android SDK Tools, revision 25.2.2
                echo "y" | android update sdk -u -a -t build-tools-25.0.1                # Android SDK Build-tools, revision 25

                if [ ! -d ${ANDROID_HOME}/extras/android/m2repository ]; then
                    echo "y" | android update sdk -u -a -t extra-android-m2repository        # Android Support Repository, revision 40
                fi
                if [ ! -d ${ANDROID_HOME}/extras/android/support ]; then
                    echo "y" | android update sdk -u -a -t extra-android-support             # Android Support Library, revision 23.2.1 (Obsolete)
                fi
                if [ ! -d ${ANDROID_HOME}/extras/google/google_play_services ]; then
                    echo "y" | android update sdk -u -a -t extra-google-google_play_services # Google Play services, revision 38
                fi
                if [ ! -d ${ANDROID_HOME}/extras/google/m2repository ]; then
                    echo "y" | android update sdk -u -a -t extra-google-m2repository         # Google Repository, revision 40
                fi

                # Downloads the constraint-layouts files from Artifactory (Updated to current version 1/12/16)
                # wget --auth-no-challenge --user=\${A_USER} --password=\${A_PASS} https://artifactory.visualtao.net/android-tmp/m2repository.tar.gz -O -| tar zfxv - -C "${ANDROID_HOME}/extras/"
            """
        }
    }

        stage('Docker image building'){
        java = docker.build 'openjdk8:android'
        java.inside("-e ANDROID_SDK_HOME=${GRADLE_USER_HOME}/android-sdk-linux -e ANDROID_HOME=${GRADLE_USER_HOME}/android-sdk-linux" ) {
            withCredentials([ // Use Jenkins credentials ID of artifactory
                [$class: 'UsernamePasswordMultiBinding', credentialsId: artifactory_creds, usernameVariable: 'A_USER', passwordVariable: 'A_PASS'],
                ]){
                    stage('Gradle Clean'){
                        sh """
                        export HOME=$GRADLE_USER_HOME
                        export JAVA_HOME="/usr/lib/jvm/java-8-openjdk-amd64"
                        ./gradlew clean
                        """
                    }

                    stage('Lint run') {
                        sh """
                            export HOME=$GRADLE_USER_HOME
                            export JAVA_HOME="/usr/lib/jvm/java-8-openjdk-amd64"
                            ./gradlew lint${BUILDFLAV}${BUILDTYPE} -x lint
                        """
                    }

                    stage('Compilation'){
                        sh """
                            export HOME=$GRADLE_USER_HOME
                            export JAVA_HOME="/usr/lib/jvm/java-8-openjdk-amd64"
                            ./gradlew compile${BUILDFLAV}${BUILDTYPE}Sources -x lint
                        """
                    }

                    stage('Unit Test') {
                        sh """
                            export HOME=$GRADLE_USER_HOME
                            export JAVA_HOME="/usr/lib/jvm/java-8-openjdk-amd64"
                            ./gradlew test${BUILDFLAV}${BUILDTYPE}UnitTest -x lint
                        """
                    }

                    stage("Assembling apk"){
                        sh """
                        export HOME=$GRADLE_USER_HOME
                        export JAVA_HOME="/usr/lib/jvm/java-8-openjdk-amd64"
                        VERSION=\$(git tag | grep '^[0-9]' | tail -1)
                        ./gradlew -DBUILD_FLAVOR=${BUILDFLAV} -DUSE_OLD_BUILD_PROCESS=false -DCORE_BRANCH=NONE -DVERSION_NAME='\$VERSION' -DBUILD_TYPE=${BUILDTYPE} -DGIT_BRANCH=origin/master -DANDROID_VIEWS_BRANCH= assemble${BUILDFLAV}${BUILDTYPE}
                        """
                    }

                    stage('Post steps'){
                        sh """
                            # Add libCore.so files to symbols.zip
                            find ${cwd}/Product-CoreSDK/obj/local -name libCore.so | zip -r ${cwd}/Product/build/outputs/symbols.zip -@

                            # Remove unaligned apk's
                            rm -f ${cwd}/Product/build/outputs/apk/*-unaligned.apk
                        """
                    }
                }
        }
    }

    stage('Test results processing'){
        check_test_results('**/build/test-results/**/*.xml')
    }

    stage('Archive artifacts'){
        if( "${BUILDFLAV}" == 'Production'){
            step([$class: 'ArtifactArchiver', artifacts: "**/build/outputs/mapping/production/release/mapping.txt", fingerprint: false])
        }
    // Archive completed apk and symbols.zip artifacts and test results
        step([$class: 'ArtifactArchiver', artifacts: '**/build/outputs/apk/*.apk', fingerprint: false])
        step([$class: 'ArtifactArchiver', artifacts: '**/build/outputs/symbols.zip', fingerprint: false])
        step([$class: 'JUnitResultArchiver', testResults: '**/build/test-results/**/*.xml', fingerprint: false])
    }

    /*
    stage('Mixpanel annotation'){
        sh """
            MP_API_KEY=${mp_api_key}
            MP_API_SECRET=${mp_api_secret}
            RELEASE_DATE=\$(date +'%Y-%m-%d %H:%M:%S')
            MP_VERSION_NAME=\$(git tag | grep '^[0-9]' | tail -n 1)
            MP_EXPIRE="1588896000"
            MP_APP_PLATFORM="Android"
            MP_BASE_URL="http://mixpanel.com/api/2.0/annotations/create?"
            MP_RELEASE_NOTES="Test by Itai"
            DESCRIPTION="\$MP_APP_PLATFORM v\$MP_VERSION_NAME \$MP_RELEASE_NOTES"
            REQUEST_URL="api_key=\$MP_API_KEY&date=\$RELEASE_DATE&description=\$DESCRIPTION&expire=\$MP_EXPIRE"
            REQUEST_URL_NO_AMPERSAND=\$(echo "\$REQUEST_URL" | tr -d '&' )
            REQUEST_URL_NO_SPACE=\$(echo "\$REQUEST_URL" | sed -e 's/ /%20/g')
            REQUEST_URL_API_SECRET="\$REQUEST_URL_NO_AMPERSAND\$MP_API_SECRET"
            SIGNATURE=\$(echo -n "\$REQUEST_URL_API_SECRET" | md5sum | awk '{print \$1}')
            CURL_COMMAND="\$MP_BASE_URL\$REQUEST_URL_NO_SPACE&sig=\$SIGNATURE"
            curl -v \$CURL_COMMAND
        """
    }
    */

    if (currentBuild.result == 'SUCCESS') {
        echo "currentBuild.result: ${currentBuild.result}"
        slackSend channel: channel, color: 'good', teamDomain: null, token: null,
            message: "*${env.JOB_NAME}* Finished Successfuly! :thumbsup: ${jlink}(<!here|here>/${ulink})"
        step([$class: 'Mailer', notifyEveryUnstableBuild: true,
              recipients: sendTo,
              sendToIndividuals: true
        ])
    }
}
