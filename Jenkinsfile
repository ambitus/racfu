def python_versions
def python_executables_and_wheels_map

pipeline {
  agent {
    node {
      label 'zOS_pyRACF_RACFu'
    }
  }

  parameters {
    // Note: Each Python version listed must be installed on the 
    // build agent and must be added to '$PATH' and '$LIBPATH'.
    string (
      name: "pythonVersions",
      defaultValue: "",
      description: (
        "(Required Always) Comma separated list of Python versions to build "
        + "wheels for (i.e., Use '12,13' for Python 3.12 and Python 3.13)."
      )
    )
    booleanParam(
      name: "createRelease",
      defaultValue: false,
      description: "Toggle whether or not to create a release from this revision."
    )
    string(
      name: "releaseTag",
      defaultValue: "",
      description: (
        "(Required When Creating Releases) This will be "
        + "the git tag and version number of the release."
      )
    )
    string(
      name: "gitHubMilestoneLink",
      defaultValue: "",
      description: (
        "(Required When Creating Releases) This is the GitHub "
        + "Milestone URL that coresponds to the release."
      )
    )
    booleanParam(
      name: "preRelease",
      defaultValue: true,
      description: "Toggle whether or not this is a pre-release."
    )
  }

  options {
    ansiColor('css')
  }

  environment {
    ZOPEN_ROOTFS = "${env.WORKSPACE}/zopen/zopen"
  }

  stages {
    stage('Parameter Validation') {
      steps {
        script {
          if (params.pythonVersions == "") {
            error("'pythonVersions' is required parameter.")
          }
          if (params.createRelease) {
            if (params.releaseTag == "") {
                error("'releaseTag' is a required parameter when creating a release.")
            }
            if (params.gitHubMilestoneLink == "") {
                error("'gitHubMilestoneLink' is a required parameter when creating a release.")
            }
          }
        }
      }
    }
    stage('Prepare') {
      steps {
        clean_python_environment()
      }
    }
    stage('Get ZOSLIB and OpenSSL from zopen community') {
      steps {
        script {
          zoslib_build_id = "20250425_143120"
          openssl_version = "3.4.0"
          openssl_build_id = "20250123_021310"

          sh "mkdir zopen"

          dir('zopen') { 
            echo "Installing zoslib ..."
            sh """
                curl -o zoslib-zopen2.${zoslib_build_id}.zos.pax.Z \\
                    -L https://github.com/zopencommunity/zoslibport/releases/download/STABLE_zoslibport_3261/zoslib-zopen2.${zoslib_build_id}.zos.pax.Z 
                pax -rf zoslib-zopen2.${zoslib_build_id}.zos.pax.Z
            """
            echo "Installing OpenSSL"
            sh """
                curl -o openssl-${openssl_version}.${openssl_build_id}.zos.pax.Z \\
                    -L https://github.com/zopencommunity/opensslport/releases/download/STABLE_opensslport_2838/openssl-${openssl_version}.${openssl_build_id}.zos.pax.Z
                pax -rf openssl-${openssl_version}.${openssl_build_id}.zos.pax.Z
            """
            echo "Creating symbolic links ..."
            sh """
                mkdir -p zopen/usr/local/lib
                mkdir -p zopen/usr/local/include
                ln -s ${env.WORKSPACE}/zopen/zoslib-zopen2/lib/libzoslib.a \\
                    ${env.WORKSPACE}/zopen/zopen/usr/local/lib/libzoslib.a
                ln -s ${env.WORKSPACE}/zopen/openssl-${openssl_version}/lib/libcrypto.a \\
                    ${env.WORKSPACE}/zopen/zopen/usr/local/lib/libcrypto.a
                ln -s ${env.WORKSPACE}/zopen/openssl-${openssl_version}/lib/libssl.a \\
                    ${env.WORKSPACE}/zopen/zopen/usr/local/lib/libssl.a
                ln -s ${env.WORKSPACE}/zopen/openssl-${openssl_version}/include/openssl \\
                    ${env.WORKSPACE}/zopen/zopen/usr/local/include/openssl
            """
          }
        }
      }
    }
    stage('Cppcheck') {
      steps {
        echo "Running cppcheck ..."
        sh "gmake check"
      }
    }
    stage('Unit Test') {
      steps {
        echo "Running unit tests ..."
        sh "gmake test"
      }
    }
    stage('Create Python Distribution Metadata') {
      when { 
        expression { params.createRelease == true }    
      }
      steps {
        script {
          python_versions = params.pythonVersions.split(",")
          python_executables_and_wheels_map = (
            create_python_executables_and_wheels_map(python_versions)
          )
        }
      }
    }
    stage('Publish') {
      when { 
        expression { params.createRelease == true }    
      }
      steps {
        publish(
          python_executables_and_wheels_map,
          params.releaseTag, 
          env.BRANCH_NAME, 
          params.gitHubMilestoneLink,
          params.preRelease
        )
      }
    }
  }
  post {
    always {
      echo "Cleaning up workspace ..."
      cleanWs()
      clean_python_environment()
    }
  }
}

def create_python_executables_and_wheels_map(python_versions) {
  def os = sh(
    returnStdout: true, 
    script: "uname"
  ).trim().replace("/", "").toLowerCase()
  def zos_release = sh(
    returnStdout: true, 
    script: "uname -r"
  ).trim().replace(".", "_")
  def processor = sh(
    returnStdout: true, 
    script: "uname -m"
  ).trim()
  def racfu_version = sh(
    returnStdout: true, 
    script: "cat pyproject.toml | grep version | cut -d'=' -f2 | cut -d'\"' -f2"
  ).trim()

  python_executables_and_wheels_map = [:]

  for (version in python_versions) {
    python_executables_and_wheels_map["python3.${version}"] = [
      "wheelDefault": (
        "racfu-${racfu_version}-cp3${version}-cp3${version}-${os}_${zos_release}_${processor}.whl"
      ),
      "wheelPublish": "racfu-${racfu_version}-cp3${version}-none-any.whl",
      "tarPublish": "racfu-${racfu_version}.tar.gz"
    ]
  }

  return python_executables_and_wheels_map
}

def clean_python_environment() {
  echo "Cleaning Python environment ..."

  sh """
    rm -rf ~/.cache
    rm -rf ~/.local
  """
}

def publish(
    python_executables_and_wheels_map, 
    release, 
    git_branch, 
    milestone, 
    pre_release
) {
  if (pre_release == true) {
    pre_release = "true"
  }
  else {
    pre_release = "false"
  }
  withCredentials(
    [
      usernamePassword(
        credentialsId: 'pyracf-racfu-github-access-token',
        usernameVariable: 'github_user',
        passwordVariable: 'github_access_token'
      )
    ]
  ) {

    // Creating GitHub releases: 
    // https://docs.github.com/en/rest/releases/releases?apiVersion=2022-11-28#create-a-release
    // Uploading release assets: 
    // https://docs.github.com/en/rest/releases/assets?apiVersion=2022-11-28#upload-a-release-asset

    // Use single quotes for credentials since it is the most secure
    // method for interpolating secrets according to the Jenkins docs:
    // https://www.jenkins.io/doc/book/pipeline/jenkinsfile/#string-interpolation

    echo "Creating '${release}' GitHub release ..."

    def description = build_description(python_executables_and_wheels_map, release, milestone)

    def release_id = sh(
      returnStdout: true,
      script: (
        'curl -f -v -L '
        + '-X POST '
        + '-H "Accept: application/vnd.github+json" '
        + '-H "Authorization: Bearer ${github_access_token}" '
        + '-H "X-GitHub-Api-Version: 2022-11-28" '
        + "https://api.github.com/repos/ambitus/racfu/releases "
        + "-d '{"
        + "     \"tag_name\": \"${release}\","
        + "     \"target_commitish\": \"${git_branch}\","
        + "     \"name\": \"${release}\","
        + "     \"body\": \"${description}\","
        + "     \"draft\": false,"
        + "     \"prerelease\": ${pre_release},"
        + "     \"generate_release_notes\":false"
        + "}' | grep '\"id\": ' | head -n1 | cut -d':' -f2 | cut -d',' -f1"
      )
    ).trim()

    def checksums_file = "SHASUMS256.txt.asc"

    sh """
      touch dist/${checksums_file}
      chtag -t -c ISO8859-1 dist/${checksums_file}
    """
    
    def tar_publish
    def tar_published = false

    echo "Cleaning repo ..."
    sh "git clean -fdx -e zopen"

    for (python in python_executables_and_wheels_map.keySet()) {
      def wheel_default = python_executables_and_wheels_map[python]["wheelDefault"]
      def wheel_publish = python_executables_and_wheels_map[python]["wheelPublish"]

      echo "Building '${wheel_default}' ..."

      sh """
        ${python} -m pip install build>=1.2.2
        ${python} -m build -w
        mv dist/${wheel_default} dist/${wheel_publish}
      """

      if (tar_published == false) {
        tar_publish = python_executables_and_wheels_map[python]["tarPublish"]
        echo "Building '${tar_publish}' ..."
        sh "${python} -m build -s"
        
        tar_published = true
      }

      echo "Uploading '${wheel_default}' as '${wheel_publish}' to '${release}' GitHub release ..."
      upload_asset(release_id, wheel_publish)

      echo "Adding sha256 checksum for '${wheel_publish}' to ${checksums_file}..."
      sh "cd dist && sha256sum -t ${wheel_publish} >> ${checksums_file}"
    }

    echo "Uploading '${tar_publish}' to '${release}' GitHub release ..."
    upload_asset(release_id, tar_publish)

    echo "Adding sha256 checksum for '${tar_publish}' to ${checksums_file}..."
    sh "cd dist && sha256sum -t ${tar_publish} >> ${checksums_file}"

    echo "Uploading '${checksums_file}' to '${release}' GitHub release ..."
    upload_asset(release_id, checksums_file)
  }
}

def upload_asset(release_id, release_asset) {
  sh(
    'curl -f -v -L '
    + '-X POST '
    + '-H "Accept: application/vnd.github+json" '
    + '-H "Authorization: Bearer ${github_access_token}" '
    + '-H "X-GitHub-Api-Version: 2022-11-28" '
    + '-H "Content-Type: application/octet-stream" '
    + "\"https://uploads.github.com/repos/ambitus/racfu/releases/${release_id}/assets?name=${release_asset}\" "
    + "--data-binary \"@dist/${release_asset}\""
  )
}

def build_description(python_executables_and_wheels_map, release, milestone) {
  def description = "Release Milestone: ${milestone}\\n&nbsp;\\n&nbsp;\\n"

  for (python in python_executables_and_wheels_map.keySet()) {
    def wheel = python_executables_and_wheels_map[python]["wheelPublish"]
    def python_executable = python
    def python_label = python.replace("python", "Python ")
    description += (
      "Install From **${python_label} Wheel Distribution** *(pre-built)*:\\n"
      + "```\\ncurl -O -L https://github.com/ambitus/racfu/releases/download/${release}/${wheel} "
      + "&& ${python_executable} -m pip install ${wheel}\\n```\\n"
    )
  }

  def python = python_executables_and_wheels_map.keySet()[-1]
  def tar = python_executables_and_wheels_map[python]["tarPublish"]
  description += (
    "Install From **Source Distribution** *(build on install)*:\\n"
    + "> :warning: _Requires z/OS Open XL C/C++ 2.1 compiler._\\n"
    + "```\\ncurl -O -L https://github.com/ambitus/racfu/releases/download/${release}/${tar} "
    + "&& python3 -m pip install ${tar}\\n```\\n"
  )

  return description
}
