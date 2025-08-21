// --- Helper to run the same command on Linux (sh) or Windows (bat) ---
def run(String cmd) {
  if (isUnix()) { sh label: 'sh', script: cmd }
  else { bat label: 'bat', script: cmd }
}

pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
    durabilityHint('PERFORMANCE_OPTIMIZED')
    // keeps console smaller; raise if you need more logs
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  environment {
    // Speed up and quiet down pip
    PIP_DISABLE_PIP_VERSION_CHECK = '1'
    PIP_NO_PYTHON_WARNINGS = '1'
    PIP_CACHE_DIR = "${WORKSPACE}/.pip-cache"
    VENV_DIR = '.venv'
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Set up Python') {
      steps {
        script {
          // Prefer python3 on *nix, python on Windows
          def sysPy = isUnix() ? 'python3' : 'python'
          run("${sysPy} -m venv ${env.VENV_DIR}")

          // Compute venv paths for portability and reuse later
          env.VENV_BIN = isUnix() ? "${env.VENV_DIR}/bin" : "${env.VENV_DIR}\\Scripts"
          env.PY       = isUnix() ? "${env.VENV_BIN}/python" : "${env.VENV_BIN}\\python.exe"
          env.PIP      = isUnix() ? "${env.VENV_BIN}/pip"    : "${env.VENV_BIN}\\pip.exe"

          // Upgrade core tooling
          run("${env.PY} -m pip install -U pip setuptools wheel")
        }
      }
    }

    stage('Install dependencies') {
      steps {
        script {
          if (fileExists('requirements.txt')) {
            run("${env.PIP} install -r requirements.txt")
          } else {
            echo "No requirements.txt found — skipping dependency install."
          }
        }
      }
    }

    stage('Static checks (lint & security)') {
      steps {
        script {
          run("${env.PIP} install flake8 bandit")
          // Create report folders
          run(isUnix() ? "mkdir -p reports/junit reports/logs" : "mkdir reports && mkdir reports\\junit && mkdir reports\\logs")
          // Lint (fast error checks then full stats). flake8 doesn’t emit JUnit by default, so we keep logs.
          run("${env.VENV_BIN}${isUnix()?'/':''}flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics > reports/logs/flake8-critical.txt || exit /b 0")
          run("${env.VENV_BIN}${isUnix()?'/':''}flake8 . --count --max-complexity=12 --max-line-length=120 --statistics > reports/logs/flake8-full.txt || exit /b 0")
          // Security scan
          run("${env.VENV_BIN}${isUnix()?'/':''}bandit -r . -q -f txt -o reports/logs/bandit.txt || exit /b 0")
        }
      }
    }

    stage('Test & coverage') {
      steps {
        script {
          run("${env.PIP} install pytest pytest-cov")
          // JUnit + Cobertura-style coverage.xml (coverage.py)
          run("""
${env.PY} -m pytest -q \\
  --junitxml=reports/junit/pytest-results.xml \\
  --cov=. --cov-report=xml:coverage.xml --cov-report=term
""".stripIndent())
        }
      }
    }

    stage('Build package') {
      when {
        anyOf {
          expression { fileExists('pyproject.toml') }
          expression { fileExists('setup.py') }
        }
      }
      steps {
        script {
          // Prefer 'build' (PEP 517). Fallback to setup.py if needed.
          run("${env.PIP} install build || ${env.PY} -m pip install build")
          run("${env.PY} -m build")
        }
      }
    }
  }

  post {
    always {
      // Publish test results (will mark build unstable if there are failures)
      junit testResults: 'reports/junit/*.xml', allowEmptyResults: true

      // Publish coverage if you have the "Coverage" plugin installed.
      // If you use the older "Cobertura" plugin, replace with: cobertura coberturaReportFile: 'coverage.xml', onlyStable: false
      script {
        try {
          publishCoverage adapters: [coberturaAdapter('coverage.xml')], sourceFileResolver: sourceFiles('STORE_LAST_BUILD')
        } catch (ignored) {
          echo "Coverage plugin not installed or misconfigured — skipped publishCoverage."
        }
      }

      // Archive build outputs and logs
      archiveArtifacts artifacts: 'dist/**/*, reports/**/*, coverage.xml, htmlcov/**', fingerprint: true, allowEmptyArchive: true
    }
    success {
      echo '✅ Python pipeline completed successfully.'
    }
    unstable {
      echo '⚠️ Build is unstable (likely test failures or low coverage).'
    }
    failure {
      echo '❌ Build failed. Check preceding stage logs.'
    }
  }
}
