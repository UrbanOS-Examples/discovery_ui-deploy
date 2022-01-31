library(
    identifier: 'pipeline-lib@4.8.0',
    retriever: modernSCM([$class: 'GitSCMSource',
                          remote: 'https://github.com/SmartColumbusOS/pipeline-lib',
                          credentialsId: 'jenkins-github-user'])
)

properties([
    pipelineTriggers([scos.dailyBuildTrigger()]),
    parameters([
        booleanParam(defaultValue: false, description: 'Deploy to development environment?', name: 'DEV_DEPLOYMENT'),
        string(defaultValue: 'development', description: 'Image tag to deploy to dev environment', name: 'DEV_IMAGE_TAG')
    ])
])

def doStageIf = scos.&doStageIf
def doStageIfDeployingToDev = doStageIf.curry(env.DEV_DEPLOYMENT == "true")
def doStageIfMergedToMaster = doStageIf.curry(scos.changeset.isMaster && env.DEV_DEPLOYMENT == "false")
def doStageIfRelease = doStageIf.curry(scos.changeset.isRelease)

node ('infrastructure') {
    ansiColor('xterm') {
        scos.doCheckoutStage()

        doStageIfDeployingToDev('Deploy to Dev') {
            deployTo(environment: 'dev', extraArgs: "--recreate-pods --set image.tag=${env.DEV_IMAGE_TAG} --values=discovery-dev.yaml")
        }

        doStageIfMergedToMaster('Process Dev job') {
            scos.devDeployTrigger('discovery_ui', 'development', 'smartcolumbusos')
        }

        doStageIfMergedToMaster('Deploy to Staging') {
            deployTo(environment: 'staging', extraArgs: "--values=discovery-staging.yaml")
            scos.applyAndPushGitHubTag('staging')
        }

        doStageIfRelease('Deploy to Production') {
            deployTo(environment: 'prod', internal: false, extraArgs: "--values=discovery-prod.yaml")
            scos.applyAndPushGitHubTag('prod')
        }
    }
}

def deployTo(params = [:]) {
    def environment = params.get('environment')
    def internal = params.get('internal', true)
    def extraArgs = params.get('extraArgs', '')
    if (environment == null) throw new IllegalArgumentException("environment must be specified")

    scos.withEksCredentials(environment) {
        def terraformOutputs = scos.terraformOutput(environment)
        def subnets = terraformOutputs.public_subnets.value.join(/\\,/)
        def allowInboundTrafficSG = terraformOutputs.allow_all_security_group.value
        def certificateARNs = [terraformOutputs.root_tls_certificate_arn.value,terraformOutputs.tls_certificate_arn.value].join(/\\,/)
        def ingressScheme = internal ? 'internal' : 'internet-facing'
        def dnsZone = terraformOutputs.internal_dns_zone_name.value
        def rootDnsZone = terraformOutputs.root_dns_zone_name.value
        def rootWAFACLARN = terraformOutputs.eks_cluster_waf_acl_arn.value


        sh("""#!/bin/bash
            set -xe

            helm repo add scdp https://urbanos-public.github.io/charts
            helm repo update
            helm upgrade --install discovery-ui scdp/discovery-ui \
                --version 1.0.1 \
                --namespace=discovery \
                --values=discovery-base.yaml \
                ${extraArgs} \
                --set global.ingress.annotations."alb\\.ingress\\.kubernetes\\.io\\/scheme"="${ingressScheme}" \
                --set global.ingress.annotations."alb\\.ingress\\.kubernetes\\.io\\/subnets"="${subnets}" \
                --set global.ingress.annotations."alb\\.ingress\\.kubernetes\\.io\\/security\\-groups"="${allowInboundTrafficSG}" \
                --set global.ingress.dns_zone="${dnsZone}" \
                --set global.ingress.root_dns_zone="${rootDnsZone}" \
                --set global.ingress.annotations."alb\\.ingress\\.kubernetes\\.io\\/wafv2-acl-arn"="${rootWAFACLARN}" \
                --set global.ingress.annotations."alb\\.ingress\\.kubernetes\\.io\\/certificate-arn"="${certificateARNs}" \
                --set image.environment="${environment}"
        """.trim())
    }
}