def label = "ewallet-${UUID.randomUUID().toString()}"
def yamlSpec = """
spec:
  nodeSelector:
    cloud.google.com/gke-preemptible: "true"
  tolerations:
    - key: dedicated
      operator: Equal
      value: worker
      effect: NoSchedule
"""

podTemplate(
    label: label,
    yaml: yamlSpec,
    containers: [
        containerTemplate(
            name: 'jnlp',
            image: 'omisegoimages/jenkins-slave:3.19-alpine',
            args: '${computer.jnlpmac} ${computer.name}',
            resourceRequestCpu: '200m',
            resourceLimitCpu: '500m',
            resourceRequestMemory: '128Mi',
            resourceLimitMemory: '512Mi',
            envVars: [
                containerEnvVar(key: 'DOCKER_HOST', value: 'tcp://localhost:2375')
            ]
        ),
        containerTemplate(
            name: 'dind',
            image: 'docker:18.05-dind',
            privileged: true,
            resourceRequestCpu: '500m',
            resourceLimitCpu: '1000m',
            resourceRequestMemory: '512Mi',
            resourceLimitMemory: '1024Mi',
        ),
    ],
) {
    node(label) {
        Random random = new Random()

        def workDir = pwd()
        def tmpDir = pwd(tmp: true)

        def project = 'gcr.io/omise-go'
        def appName = 'ewallet'
        def imageName = "${project}/${appName}"
        def releaseVersion = '0.1.0-beta'

        def nodeIP = getNodeIP()
        def gitCommit

        stage('Checkout') {
            checkout scm
            gitCommit = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
        }

        stage('Test') {
            docker.image('postgres:9.6.9-alpine').withRun('-p 5432:5432') { c ->
                docker.image('postgres:9.6.9-alpine').inside {
                    sh("pg_isready -t 60 -h ${nodeIP} -p 5432")
                }

                docker.image('elixir:1.6.5-alpine').inside([
                        "-e MIX_ENV=test",
                        "-e DATABASE_URL=postgresql://postgres@${nodeIP}:5432/ewallet_${gitCommit}_ewallet",
                        "-e LOCAL_LEDGER_DATABASE_URL=postgresql://postgres@${nodeIP}:5432/ewallet_${gitCommit}_local_ledger",
                    ].join(' ')
                ) {
                    sh("""
                        apk add --update --no-cache git build-base libsodium-dev imagemagick
                        mix do local.hex --force, local.rebar --force
                        mix do format --check-formatted
                        mix do deps.get, compile
                        mix do credo, ecto.create, ecto.migrate
                        mix test
                    """.stripIndent())
                }
            }
        }
    }
}

String getNodeIP() {
    def rawNodeIP = sh(script: 'ip -4 -o addr show scope global', returnStdout: true).trim()
    def matched = (rawNodeIP =~ /inet (\d+\.\d+\.\d+\.\d+)/)
    return "" + matched[0].getAt(1)
}

String getPodID(String opts) {
    def pods = sh(script: "kubectl get pods ${opts} -o name", returnStdout: true).trim()
    def matched = (pods.split()[0] =~ /pods\/(.+)/)
    return "" + matched[0].getAt(1)
}
