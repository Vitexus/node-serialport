#!groovy

// Current version of this Pipeline https://github.com/VitexSoftware/BuildImages/blob/main/Test/Jenkinsfile-parael

// debian:forky disabled: Forky is still unstable/research-only.
// The full Debian package ecosystem is not yet available for Forky.
// Re-enable once the stack builds cleanly for Forky.
String[] distributions = ['debian:bookworm', 'debian:trixie', 'ubuntu:jammy', 'ubuntu:noble']

String vendor = 'vitexsoftware'
//String distroFamily = ''

properties([
    copyArtifactPermission('Foregin/*')
])
node() {
    ansiColor('xterm') {
        stage('SCM Checkout') {
            checkout scm
        }
    }
}

def branches = [:]
distributions.each { distro ->
    branches[distro] = {
        def distroName = distro
        println  "Dist:" + distroName

        def dist = distroName.split(':')
        def distroFamily = dist[0]
        def distroCode = dist[1]
        def buildImage = ''
        def artifacts = []
        def buildVer = ''

        node {
            ansiColor('xterm') {
                stage('Checkout ' + distroName) {
                    checkout scm
                    def imageName = vendor + '/' + distro
                    buildImage = docker.image(imageName)
                    sh 'git checkout debian/changelog'
                    def version = sh (
                        script: 'dpkg-parsechangelog --show-field Version',
                        returnStdout: true
                    ).trim()
                    buildVer = version + '.' + env.BUILD_NUMBER  + '~' + distroCode
                }
                stage('Build ' + distroName) {
                    buildImage.inside {
                        withEnv(["BUILDVER=${buildVer}"]) {
                            sh 'dch -b -v "$BUILDVER" "$BUILD_TAG"'
                        }
                        sh 'sudo apt-get update --allow-releaseinfo-change'
                        sh 'sudo chown jenkins:jenkins ..'
                        sh 'debuild-pbuilder -r"sudo -E" -i -us -uc -b'
                        sh '''
                            mkdir -p $WORKSPACE/dist/debian/
                            rm -rf $WORKSPACE/dist/debian/*
                            while IFS= read -r deb _; do
                                case "$deb" in -*) continue;; esac
                                [ -n "$deb" ] || continue
                                mv -- "../$deb" "$WORKSPACE/dist/debian/"
                            done < debian/files
                        '''
                        artifacts = sh (
                            script: "cat debian/files | awk '{print \$1}'",
                            returnStdout: true
                        ).trim().split('\n')
                    }
                }

                stage('Test ' + distroName) {
                    buildImage.inside {
                        def debconf_debug = 0 //Set to "5" or "developer" to debug debconf
                        sh 'cd $WORKSPACE/dist/debian/ ; dpkg-scanpackages . /dev/null > Packages; gzip -9c Packages > Packages.gz; cd $WORKSPACE'
                        sh 'echo "deb [trusted=yes] file://///$WORKSPACE/dist/debian/ ./" | sudo tee /etc/apt/sources.list.d/local.list'
                        sh 'sudo apt-get update --allow-releaseinfo-change'
                        sh 'echo "INSTALATION"'

                    def installOrder = [
                        'node-serialport',
                    ]

                    def sorted_artifacts = artifacts.toList()
                    installOrder.each { pkgPrefix ->
                        def debFile = null
                        for (item in sorted_artifacts) {
                            def itemStr = item.toString()
                            if (itemStr.startsWith(pkgPrefix) && itemStr.endsWith('.deb')) {
                                debFile = itemStr
                                break
                            }
                        }
                        if (debFile) {
                            def pkgName = debFile.tokenize('/')[-1].replaceFirst(/_.*/, '')
                            if (!(pkgName ==~ /^[a-z0-9][a-z0-9+.\-]+$/)) { error("Invalid package name: ${pkgName}") }
                            withEnv(["PKG=${pkgName}"]) {
                                sh 'echo "Installing $PKG on $(lsb_release -sc)"'
                                sh 'sudo DEBIAN_FRONTEND=noninteractive apt-get -y install "$PKG"'
                            }
                        }
                    }
                    sh 'sudo rm -f /etc/apt/sources.list.d/local.list'

                }
                stage('Archive artifacts ' + distroName ) {
                    // Only run if previous stages (Build and Test) succeeded
                    buildImage.inside {
                        // Archive all produced artifacts listed in debian/files
                        artifacts.each { deb_file ->
                            println "Archiving artifact: " + deb_file
                            archiveArtifacts artifacts: 'dist/debian/' + deb_file
                        }

                        // Cleanup: remove any produced files named in debian/files
                        sh '''
                            set -e
                            if [ -f debian/files ]; then
                              while read -r file _; do
                                [ -n "$file" ] || continue
                                rm -f "dist/debian/$file" || true
                                rm -f "../$file" || true
                                rm -f "$WORKSPACE/$file" || true
                              done < debian/files
                            fi
                        '''
                    }
                }
            }
        }
    }
}
}

parallel branches

node {
    stage('Publish to Aptly') {
	publishDebToAptly()
    }
}
