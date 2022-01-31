library(
    identifier: 'pipeline-lib@4.6.1',
    retriever: modernSCM([$class: 'GitSCMSource',
                          remote: 'https://github.com/SmartColumbusOS/pipeline-lib',
                          credentialsId: 'jenkins-github-user'])
)

node('infrastructure') {
    scos.doCheckoutStage()
    stage("Scan Discovery") {
        docker.image('owasp/zap2docker-stable:latest').inside {
            sh """
                mkdir /zap/wrk
                cp zap.conf /zap/wrk
            """
            sh """
                zap-full-scan.py \
                -t https://discovery.staging-smartos.com \
                -c zap.conf
            """
        }
    }
}
