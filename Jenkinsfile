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

    stage ('upload apk....'){
            try {
                echo "上传制品中...."
                sh """
                ls
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