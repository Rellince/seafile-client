pipeline {
  agent { label 'linux' }

  options {
    timeout(time: 45, unit: 'MINUTES')
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '5'))
  }

  triggers {
    pollSCM('H/5 * * * *')
  }

  environment {
    SHA_SHORT = "${env.GIT_COMMIT?.take(12) ?: 'manual'}"
    VERSION   = "rellince-${env.BUILD_NUMBER}-${SHA_SHORT}"
  }

  stages {
    stage('checkout') {
      steps { checkout scm }
    }

    stage('build linux') {
      steps {
        // Whole build runs inside an Ubuntu 22.04 sibling container so the apt
        // deps + the from-source libsearpc/libseafile install don't pollute the
        // ephemeral Jenkins agent. Output (AppImage) lands in $WORKSPACE.
        sh '''
          set -e
          docker run --rm \\
            -v $WORKSPACE:/work -w /work \\
            -e DEBIAN_FRONTEND=noninteractive \\
            -e VERSION="$VERSION" \\
            --user 0:0 \\
            ubuntu:22.04 \\
            bash -ec "
              apt-get update
              apt-get install -y --no-install-recommends \\
                build-essential cmake intltool valac libtool autoconf automake pkg-config re2c flex \\
                qtbase5-dev qttools5-dev qttools5-dev-tools libqt5webengine5 qtwebengine5-dev \\
                libssl-dev libcurl4-openssl-dev libsqlite3-dev libglib2.0-dev \\
                libevent-dev uuid-dev libjansson-dev libarchive-dev \\
                libfuse-dev libsasl2-dev libldap2-dev libonig-dev libxml2-dev libjwt-dev libhiredis-dev \\
                sqlite3 file desktop-file-utils libfuse2 \\
                git curl ca-certificates wget

              # libsearpc + libseafile from source via haiwen's canonical script
              git clone --depth=1 --branch=master https://github.com/haiwen/seafile-test-deploy /tmp/seafile-test-deploy
              cd /tmp/seafile-test-deploy
              export JWT_PRIVATE_KEY=ci-placeholder SITE_ROOT=/ \\
                     SEAFILE_MYSQL_DB_CCNET_DB_NAME=ccnet \\
                     SEAFILE_MYSQL_DB_SEAFILE_DB_NAME=seafile \\
                     SEAFILE_MYSQL_DB_SEAHUB_DB_NAME=seahub
              ./bootstrap.sh
              ldconfig

              cd /work
              cmake -B build -DCMAKE_BUILD_TYPE=Release .
              cmake --build build -j\\$(nproc)

              # Package AppImage
              DESTDIR=/work/AppDir cmake --install build
              cd /work
              wget -q -O linuxdeploy https://github.com/linuxdeploy/linuxdeploy/releases/download/continuous/linuxdeploy-x86_64.AppImage
              wget -q -O linuxdeploy-plugin-qt https://github.com/linuxdeploy/linuxdeploy-plugin-qt/releases/download/continuous/linuxdeploy-plugin-qt-x86_64.AppImage
              chmod +x linuxdeploy linuxdeploy-plugin-qt
              # linuxdeploy needs FUSE; --appimage-extract-and-run avoids that requirement in containers
              ./linuxdeploy --appimage-extract-and-run --appdir AppDir --plugin qt --output appimage --desktop-file AppDir/usr/share/applications/seafile.desktop
              mv Seafile_Client*.AppImage \"seafile-client-${VERSION}-x86_64.AppImage\" 2>/dev/null \\
                || mv *.AppImage \"seafile-client-${VERSION}-x86_64.AppImage\"
              chmod 644 \"seafile-client-${VERSION}-x86_64.AppImage\"
            "
        '''
      }
    }

    stage('archive') {
      steps {
        archiveArtifacts artifacts: "seafile-client-${VERSION}-x86_64.AppImage", fingerprint: true
      }
    }
  }

  post {
    always { cleanWs() }
    failure { echo "Build ${env.BUILD_NUMBER} failed: ${env.BUILD_URL}" }
  }

  // macOS & Windows builds deferred until a relative actually asks for native
  // desktop on those platforms.
}
