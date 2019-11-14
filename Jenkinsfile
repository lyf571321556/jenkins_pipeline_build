node('master') {

    def BUILD_VERSION = version()
    if (BUILD_VERSION) {
        echo "Building version ${BUILD_VERSION}"
        }
	def GIT_REVISION = GIT_Revision()
    if (GIT_REVISION) {
        echo "GIT_REVISION: ${GIT_REVISION}"
        }

    stage ('git clone code....'){
        try {
            echo "打印项目版本号：${BUILD_VERSION}"
            echo '清空构建目录：'
            sh """
            ls
            rm -rf *
            ls
            """
			echo '代码下载开始：'
            checkout([$class: 'GitSCM', branches: [[name: 'refs/tags/${ONES_TAG}']], doGenerateSubmoduleConfigurations: false, submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'ones-ai-android', url: 'https://github.com/lyf571321556/jenkins_pipeline_build.git']]])
		}
        catch (exc) {
            echo '代码下载失败了, 请检查配置！'
            emailext body: "${env.EmailextBody_Failed}",
			subject: "${JOB_NAME} - 版本${BUILD_VERSION}.${BUILD_NUMBER} - Failure!",
			to: "${params.FailedToMailList}"
            sh 'exit 1'
        }
    }

    echo "branch:${env.branch}"
    echo "branch:${env.buildingTag}"
    echo "branch:${env.BRANCH_NAME}"
    echo "current SHA:${env.GIT_COMMIT}"
    echo "previous SHA: ${env.GIT_PREVIOUS_SUCCESSFUL_COMMIT}"

    def cwd = "${JENKINS_HOME}"
    stage('Environement preparation'){
     // Build parameters
     NDK_VER="r12b"
     SDK_VER="r24.4.1"
     GRADLE_USER_HOME="$cwd/caches/.gradle"
     NDK_DIR="${GRADLE_USER_HOME}/android-ndk-${NDK_VER}"
     ANDROID_HOME="${GRADLE_USER_HOME}/android-sdk-linux"
     SDK_TOOLS="${ANDROID_HOME}/tools"
     AAPT="${ANDROID_HOME}/build-tools/28.0.3"
     withCredentials([ // Use Jenkins credentials ID of artifactory
            [$class: 'UsernamePasswordMultiBinding', credentialsId: 'ones-ai-android',usernameVariable: 'username', passwordVariable: 'password'],
        ]){
            sh """
                export PATH="\$PATH:${GRADLE_USER_HOME}:${NDK_DIR}:${ANDROID_HOME}:${SDK_TOOLS}:${SDK_TOOLS}/bin:${AAPT}"

                # Checking that Android SDK and NDK are installed
                # rm -rf ${GRADLE_USER_HOME}
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

                # Configuring path to ndk and sdk dir - as required by Gradle
                printf "ndk.dir=$NDK_DIR\nsdk.dir=$ANDROID_HOME" > "$cwd/local.properties"
                cp "$cwd/local.properties" "$cwd/Product-AndroidSDK"
                cp "$cwd/local.properties" "$cwd/Product-ServicesSDK"
                cp "$cwd/local.properties" "$cwd/Product-CoreSDK"

                # Creates licenses folder which will include the signed licenses for google sdk packages
                mkdir -p "${ANDROID_HOME}/licenses"

                echo -e "\n8933bad161af4178b1185d1a37fbf41ea5269c55\nd56f5187479451eabf01fb78af6dfcb131a6481e\n24333f8a63b6825ea9c5514f83c2829b004d1fee" > "${ANDROID_HOME}/licenses/android-sdk-license"

                echo -e "\n84831b9409646a918e30573bab4c9c91346d8abd" > "${ANDROID_HOME}/licenses/android-sdk-preview-license"

                echo -e "\nd975f751698a77b662f1254ddbeed3901e976f5a" > "${ANDROID_HOME}/licenses/intel-android-extra-license"


                mkdir -p "${ANDROID_HOME}/.android"

                touch "${ANDROID_HOME}/.android/repositories.cfg"

                # sdkmanager --update && yes | sdkmanager --licenses
                # echo "y" | android --update
                # echo "y" | android --licenses

                cat "${ANDROID_HOME}/licenses/android-sdk-license"
                cat "${ANDROID_HOME}/licenses/android-sdk-preview-license"

                # Mandatory manual downloads of required SDK tools
                echo "y" | android update sdk -u -a -t platform-tools                    # Android SDK Platform-tools, revision 29.0.5
                echo "y" | android update sdk -u -a -t tools                             # Android SDK Tools, revision 26.1.1
                echo "y" | android update sdk -u -a -t build-tools-28.0.3               # Android SDK Build-tools, revision 28.0.3

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


                 cat "${ANDROID_HOME}/licenses/android-sdk-license"
                 cat "${ANDROID_HOME}/licenses/android-sdk-preview-license"
            """
        }
    }

            BUILDFLAV = "default"
            BUILDTYPE = "Debug"
 stage('start container'){
        docker.image("airdock/oraclejdk:1.8").inside("-e ANDROID_SDK_HOME=${GRADLE_USER_HOME}/android-sdk-linux -e ANDROID_HOME=${GRADLE_USER_HOME}/android-sdk-linux" ) {
            withCredentials([ // Use Jenkins credentials ID of artifactory
                [$class: 'UsernamePasswordMultiBinding', credentialsId: 'ones-ai-android',usernameVariable: 'username', passwordVariable: 'password'],
                ]){
                    stage('build apk'){
                        sh """
                        export HOME=$GRADLE_USER_HOME
                        export GRADLE_HOME=$GRADLE_USER_HOME
                        # export JAVA_HOME="/srv/java/jdk"
                        pwd
                        java -version
                        ./gradlew assembleRelease
                        ls
                        cd app/build/outputs
                        pwd
                        ls -a
                        """
                    }

                }
        }
    }

    stage ('github-release upload'){
                try {
                    echo "上传制品中...."
                    sh """
                    export GITHUB_TOKEN='7fc7e6d5a72c0cbcf2a21c3a74d6ed61ddd034d2'
                    ls
                    pwd
                    # apk/release/*.apk
                    tar -zcvf 'ones_release_${ONES_TAG}_build${BUILD_VERSION}'.gz app/build/outputs/*
                    ls
                     ./github-release upload --user lyf571321556 --repo jenkins_pipeline_build --tag '${ONES_TAG}' --name 'ones_release_${ONES_TAG}_build${BUILD_VERSION}.gz' --file 'ones_release_${ONES_TAG}_build${BUILD_VERSION}'.gz
                    """
        		}
                catch (exc) {
                    echo '制品上传失败, 请检查网路环境！'
                    sh 'exit 1'
                }
            }
}
def version() {
def BUILD_REVISION=sh(returnStdout:true,script:"git rev-list HEAD --count").trim()
    echo "版本号${BUILD_REVISION}"
    BUILD_REVISION
}
def GIT_Revision() {
    def matcher2 = readFile('.git/HEAD') =~ '(.+)'
    matcher2 ? matcher2[0][1] : null
}