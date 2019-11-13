node('master') {

	echo 'check代码获取主版本号'
        checkout([$class: 'GitSCM', branches: [[name: 'master']], doGenerateSubmoduleConfigurations: false, extensions: [[$class: 'CleanBeforeCheckout']], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'ones-ai-android', url: 'https://github.com/lyf571321556/jenkins_pipeline_build.git']]])

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
			echo '代码下载开始：'
            checkout([$class: 'GitSCM', branches: [[name: 'master']], doGenerateSubmoduleConfigurations: false, submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'ones-ai-android', url: 'https://github.com/lyf571321556/jenkins_pipeline_build.git']]])
		}
        catch (exc) {
            echo '代码下载失败了, 请检查配置！'
            emailext body: "${env.EmailextBody_Failed}",
			subject: "${JOB_NAME} - 版本${BUILD_VERSION}.${BUILD_NUMBER} - Failure!",
			to: "${params.Maillist_Failed}"
            sh 'exit 1'
        }
    }

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
 stage('building int Docker'){
        docker.image("airdock/oraclejdk:1.8").inside("-e ANDROID_SDK_HOME=${GRADLE_USER_HOME}/android-sdk-linux -e ANDROID_HOME=${GRADLE_USER_HOME}/android-sdk-linux" ) {
            withCredentials([ // Use Jenkins credentials ID of artifactory
                [$class: 'UsernamePasswordMultiBinding', credentialsId: 'ones-ai-android',usernameVariable: 'username', passwordVariable: 'password'],
                ]){
                    stage('Gradle Clean'){
                        sh """
                        export HOME=$GRADLE_USER_HOME
                        export GRADLE_HOME=$GRADLE_USER_HOME
                        # export JAVA_HOME="/srv/java/jdk"
                        pwd
                        java -version
                        ./gradlew clean
                        """
                    }

                    stage('Lint run') {
                        sh """
                            export HOME=$GRADLE_USER_HOME
                            # export JAVA_HOME="/srv/java/jdk"
                            # ./gradlew lint${BUILDFLAV}${BUILDTYPE} -x lint
                            ./gradlew lint -x lint
                        """
                    }

                    stage('Compilation'){
                        sh """
                            export HOME=$GRADLE_USER_HOME
                            # export JAVA_HOME="/srv/java/jdk"
                            # ./gradlew compile${BUILDFLAV}${BUILDTYPE}Sources -x lint
                            ./gradlew compileSources -x lint
                        """
                    }

                    stage('Unit Test') {
                        sh """
                            export HOME=$GRADLE_USER_HOME
                            # export JAVA_HOME="/srv/java/jdk"
                            # ./gradlew test${BUILDFLAV}${BUILDTYPE}UnitTest -x lint
                            ./gradlew testUnitTest -x lint
                        """
                    }

                    stage("Assembling apk"){
                        sh """
                        export HOME=$GRADLE_USER_HOME
                        # export JAVA_HOME="/srv/java/jdk"
                        VERSION=\$(git tag | grep '^[0-9]' | tail -1)
                        # ./gradlew -DBUILD_FLAVOR=${BUILDFLAV} -DUSE_OLD_BUILD_PROCESS=false -DCORE_BRANCH=NONE -DVERSION_NAME='\$VERSION' -DBUILD_TYPE=${BUILDTYPE} -DGIT_BRANCH=origin/master -DANDROID_VIEWS_BRANCH= assemble${BUILDFLAV}${BUILDTYPE}
                        ./gradlew
                        """
                    }

                    stage('Post steps'){
                        sh """
                            # Add libCore.so files to symbols.zip
                            # find ${cwd}/Product-CoreSDK/obj/local -name libCore.so | zip -r ${cwd}/Product/build/outputs/symbols.zip -@

                            # Remove unaligned apk's
                            rm -f ${cwd}/Product/build/outputs/apk/*-unaligned.apk
                        """
                    }
                }
        }
    }

    stage ('构建'){
        try {
            sh 'gradle clean assembleDebug'
		}
        catch (exc) {
            echo '构建失败了, 请检查配置！'
            emailext body: "${env.EmailextBody_Failed}",
			subject: "${JOB_NAME} - 版本${BUILD_VERSION}.${BUILD_NUMBER} - Failure!",
			to: "${params.Maillist_Failed}"
            sh 'exit 1'
        }
    }

    stage('静态代码检查') {
		try {
			echo '静态代码检查开始：'
			withSonarQubeEnv('sonarqube6.5') {
				sh 'sonar-scanner ' +
				"-Dsonar.projectKey=${env.JOB_NAME} " +
				"-Dsonar.projectName=${env.JOB_NAME} " +
				'-Dsonar.projectVersion=1.0 ' +
				'-Dsonar.sources=app/src/ ' +
				'-Dsonar.java.binaries=app/build/intermediates/classes ' +
				'-Dsonar.java.libraries=app/libs/*.jar,E:/tools/android-sdk-windows/platforms/android-27/android.jar ' +
				'-Dsonar.skipDesign=true ' +
				'-Dsonar.sourceEncoding=UTF-8'
			}
		}
		catch (exc) {
            echo '静态代码检查失败了, 请检查配置！'
            emailext body: "${env.EmailextBody_Failed}",
			subject: "${JOB_NAME} - 版本${BUILD_VERSION}.${BUILD_NUMBER} - Failure!",
			to: "${params.Maillist_Failed}"
            sh 'exit 1'
        }
    }

    stage ('上传蒲公英'){
        try {
            //上传到蒲公英供邮件中的二维码下载
			sh """curl -F "file=@%WORKSPACE%%Output_Dir%" -F "buildUpdateDescription=CI构建版本号对应：${BUILD_VERSION}.%BUILD_NUMBER%" -F "_api_key=%Api_Key%" %Pgyer_URL%"""
		}
        catch (exc) {
            echo '上传蒲公英失败了, 请检查配置！'
            emailext body: "${env.EmailextBody_Failed}",
			subject: "${JOB_NAME} - 版本${BUILD_VERSION}.${BUILD_NUMBER} - Failure!",
			to: "${params.Maillist_Failed}"
            sh 'exit 1'
        }
    }

    stage ('上传Artifactory'){
        try {
			def SERVER_ID = 'Artifactory'
			def server = Artifactory.server SERVER_ID
			def uploadSpec =
			"""
			{
			"files": [
				{
					"pattern": "test2/build/outputs/apk/test2-debug.apk",
					"target": "Android-test-local/Android/${BUILD_VERSION}/${BUILD_NUMBER}/"
				}
			  ]
			}
			"""
			def buildInfo = Artifactory.newBuildInfo()
			buildInfo.env.capture = true
			buildInfo=server.upload(uploadSpec)
			server.publishBuildInfo(buildInfo)
		}
        catch (exc) {
            echo '上传Artifactory失败了, 请检查配置！'
            emailext body: "${env.EmailextBody_Failed}",
			subject: "${JOB_NAME} - 版本${BUILD_VERSION}.${BUILD_NUMBER} - Failure!",
			to: "${params.Maillist_Failed}"
            sh 'exit 1'
        }

	stage ('TAG'){
        sh """
            git tag -d release-${BUILD_VERSION}.${BUILD_NUMBER}
            git config --global user.email "qa-ci@xxxx.cn"
            git config --global user.name "qa-ci"
            git tag -a "release-${BUILD_VERSION}.${BUILD_NUMBER}" -m "CI Autobuild ${BUILD_VERSION}.${BUILD_NUMBER}" ${GIT_REVISION}
            git push origin "release-${BUILD_VERSION}.${BUILD_NUMBER}"
            """
    }
    }

	stage ('构建成功邮件通知'){
            emailext body: """<hr/>
			(本邮件是程序自动下发的，请勿回复！)<br/><hr/>

			构建成功啦，以下是本次构建信息： <br/><hr/>
			项目名称：\${PROJECT_NAME}<br/><hr/>
			版本号：${BUILD_VERSION}.${BUILD_NUMBER}<br/><hr/>

			GIT版本号：${GIT_REVISION}<br/><hr/>
			产物存放路径：<a href="https://cd.myones.net//artifactory/list/Android-test-local/Android/${BUILD_VERSION}/${BUILD_NUMBER}/test2-debug.apk">点击下载本次构建产物</a><br/><hr/>

			手机扫描二维码下载：<br/><img src="${Qrcode_URL}"></img><br/><hr/>

			触发原因：\${CAUSE}<br/><hr/>

			构建流水线详情：<a href="https://cd.myones.net//blue/organizations/jenkins/${JOB_NAME}/detail/${JOB_NAME}/${BUILD_NUMBER}/pipeline">https://cd.myones.net//blue/organizations/jenkins/${JOB_NAME}/detail/${JOB_NAME}/${BUILD_NUMBER}/pipeline</a><br/><hr/>

			控制台日志：<a href="${BUILD_URL}console">${BUILD_URL}console</a><br/><hr/>

			静测结果：<a href="https://cd.myones.net//dashboard/index/${JOB_NAME}">https://cd.myones.net//dashboard/index/${JOB_NAME}</a><br/><hr/>

			变更集:\${JELLY_SCRIPT,template="html"}<br/><hr/>""" ,
			subject: "${JOB_NAME} - 版本${BUILD_VERSION}.${BUILD_NUMBER} - Successful!",
			to: "${params.Maillist_Success}"
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