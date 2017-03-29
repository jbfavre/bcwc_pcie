def GIT_HOSTNAME='ci.jbfav.re'
def GIT_REPO_NAME=env.JOB_NAME
def REPOSITORY_EXTRA=""
def REPOSITORY_EXTRA_KEYS=""
def DEB_BUILD_OPTIONS=(skip_tests == "true" ? 'nocheck' : '' )
def ADT=(skip_tests == "true" ? 'skip' : 'internal')

// Checkout package git repository
// Must be done on master (my setup)
stage('Build source package')
    node('master'){
        // Clean workspace
        deleteDir()

        checkout changelog: false,
            poll: false,
            scm: [$class: 'GitSCM',
                branches: [[name: "${branch}"]],
                doGenerateSubmoduleConfigurations: false,
                extensions: [
                    [$class: 'RelativeTargetDirectory', relativeTargetDir: 'source']
                ],
                submoduleCfg: [],
                userRemoteConfigs: [[
                    credentialsId: '6e8f0868-0181-4e3b-9bc5-0e05c823de1c',
                    url: "git@${GIT_HOSTNAME}:${env.JOB_NAME}.git"
                ]]
            ]
            def GIT_COMMIT = sh(returnStdout: true, script: 'cd source;git rev-parse HEAD;cd ..')
            def GIT_BRANCH = "${branch}"
            def GIT_URL = "git@${GIT_HOSTNAME}:${GIT_REPO_NAME}.git"

        // Build source package
        // Required to be run on same node than previous stage because of GIT checkout
        // Git plugin does not expose its environement variable in Pipeline yet
        // Therefore we have to expose them ourselves
        withEnv([
            "GIT_COMMIT=${GIT_COMMIT}",
            "GIT_BRANCH=${GIT_BRANCH}",
            "GIT_URL=${GIT_URL}",
            "release=${release}",
            "branch=${branch}",
            "tag=${tag}",
            "distribution=${distribution}",
        ]) {
            sh '/usr/bin/generate-git-snapshot'
        }

        // Stash source package so that it can be build later on slaves
        stash name: 'source',
            includes: '*.gz,*.bz2,*.xz,*.deb,*.dsc,*.buildinfo,*.changes, lintian.txt',
            useDefaultExcludes: false
    }

// Build binary packages in parallel (i386 & amd64).
// Can be done on slave, but you need to have at least 2 slaves
//
// Currently, pipeline does not allow you to choose which slave runs
// which build. If you needed, you must add labels on slaves,
// like 'i386' and 'adm64'.
//
// Use REPOSITORY_EXTRA & REPOSITORY_EXTRA_KEYS if you need some other
// repositories for dependencies
stage name: 'Build binary packages'

    node('master'){
        deleteDir()
        unstash 'source'
        withEnv([
            "release=${release}",
            "branch=${branch}",
            "tag=${tag}",
            "distribution=${distribution}",
            "BUILD_ONLY=true",
            "SUDO_CMD=sudo",
            "DEB_BUILD_OPTIONS=${DEB_BUILD_OPTIONS}",
            "ADT=${ADT}",
            "REPOSITORY_EXTRA=${REPOSITORY_EXTRA}",
            "REPOSITORY_EXTRA_KEYS=${REPOSITORY_EXTRA_KEYS}"
        ]) {
            sh '/usr/bin/build-and-provide-package'
        }
        stash name: 'binaries',
            includes: '*.gz,*.bz2,*.xz,*.deb,*.dsc,*.buildinfo,*.changes,lintian.txt,piuparts*',
            useDefaultExcludes: false
    }

// Add source & binaries packages to APT repository
// Must be done on master (my setup)
stage('Run static tests')

    parallel piuparts: {

        node('master'){
            // Clean Workspace
            deleteDir()
            // Unstash binaries artifacts
            unstash 'binaries'

            withEnv([
                "release=${release}",
                "branch=${branch}",
                "tag=${tag}",
                "distribution=${distribution}",
                "DEB_BUILD_OPTIONS=${DEB_BUILD_OPTIONS}",
                "ADT=${ADT}",
            ]) {
                    sh 'sudo distribution=${distribution} /usr/bin/piuparts_wrapper $PWD/*.deb || true; /usr/bin/piuparts_tap piuparts.txt > piuparts.tap'
            }
            stash name: 'piuparts',
                includes: 'piuparts*',
                useDefaultExcludes: false
        }

    }, lintian: {

        node('master'){
            // Clean Workspace
            deleteDir()
            // Unstash binaries artifacts
            unstash 'binaries'

            withEnv([
                "release=${release}",
                "branch=${branch}",
                "tag=${tag}",
                "distribution=${distribution}",
                "DEB_BUILD_OPTIONS=${DEB_BUILD_OPTIONS}",
                "ADT=${ADT}",
           ]) {
                    sh ' mkdir -p report; /usr/bin/lintian-junit-report *.changes > report/lintian.xml'
            }
            stash name: 'lintian',
                includes: 'lintian.txt',
                useDefaultExcludes: false
        }
    }

stage('Add packages to repository')

    node('master'){

        // Clean Workspace
        deleteDir()

        // Unstash binaries artifacts
        unstash 'source'
        dir('binaries') {
            unstash 'binaries'
        }

        // Add packages to repo
        withEnv([
            "release=${release}",
            "branch=${branch}",
            "tag=${tag}",
            "distribution=${distribution}",
            "PROVIDE_ONLY=true",
            "REPO_CODENAME=${distribution}",
            "RELEASE_DISTRIBUTION=${distribution}"
        ]) {
            sh '/usr/bin/build-and-provide-package'
        }

        // Sync APT repository to S3
        withEnv([
            "release=${release}",
        ]) {
            sh '/usr/local/bin/repo-sync-to-s3'
        }
    }

stage('Archive artifacts')
    node('master'){
        unstash 'binaries'
        unstash 'piuparts'
        unstash 'lintian'
        archive '*.gz,*.bz2,*.xz,*.deb,*.dsc,*.changes,lintian.txt,piuparts*'
    }
