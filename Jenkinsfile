pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        // Keep the 10 most recent builds
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
    }

    environment {
	PATH="/usr/local/bin:$PATH"
	WORKON_HOME="~/venvs"
	VIRTUALENVWRAPPER_PYTHON="python3"
	OMP_NUM_THREADS="1"
    }

    stages {

        stage ("Code pull"){
            steps{
                checkout scm
            }
        }

        stage('Build environment') {
            steps {
                echo "Building virtualenv"
                sh  ''' . /usr/local/bin/virtualenvwrapper.sh
		    	# One of the command might fail, but that is normal, so
			# avoid exitting on failure for mkvirtualenv
			set +e
			mkvirtualenv ${BUILD_TAG}
                        export PY=3
                        pip$PY install --upgrade pip setuptools wheel
                        pip$PY install --only-binary=numpy,scipy numpy scipy
                        pip$PY install -r requirements.txt
                        pip$PY install pytest-cov
                        pip$PY install pylint pycodestyle
                        pip$PY install dormouse
                        
                        if [[ "$OSTYPE" == "darwin"* ]]; then
                            for i in {10..1}; do
                                if [[ `command -v gcc-${i}` ]]; then
                                    export CC=gcc-${i}
                                    export CXX=g++-${i}
                                    break
                                fi
                            done
                        else
                            export CC=gcc
                            export CXX=g++
                        fi
  
                        pip$PY install revkit --no-binary=revkit --ignore-installed
                        pip$PY install -e .
                    '''
            }
        }

	stage('Testing') {
	    parallel {
		stage('Static code metrics') {
		    steps {
			// echo "Raw metrics"
			// sh  ''' . /usr/local/bin/virtualenvwrapper.sh
			//         workon ${BUILD_TAG}
			//         radon raw --json irisvmpy > raw_report.json
			//         radon cc --json irisvmpy > cc_report.json
			//         radon mi --json irisvmpy > mi_report.json
			//         sloccount --duplicates --wide irisvmpy > sloccount.sc
			//     '''
			
			echo "Test coverage"
			sh  ''' . /usr/local/bin/virtualenvwrapper.sh
                        set +e && workon ${BUILD_TAG}
                        python3 -m pytest --cov=projectq --cov-report xml:reports/coverage.xml
                    '''
		    }
		    post{
			always{
			    step([$class: 'CoberturaPublisher',
				  autoUpdateHealth: false,
				  autoUpdateStability: false,
				  coberturaReportFile: 'reports/coverage.xml',
				  failNoReports: false,
				  failUnhealthy: false,
				  failUnstable: false,
				  maxNumberOfBuilds: 10,
				  onlyStable: false,
				  sourceEncoding: 'ASCII',
				  zoomCoverageChart: false])
			}
		    }
		}
		stage('Style check') {
		    steps {
			echo "PEP8 Style check"
			sh  ''' . /usr/local/bin/virtualenvwrapper.sh
                                set +e && workon ${BUILD_TAG}
                                mkdir -p reports
                                python3 -m pycodestyle projectq > reports/pycodestyle.log || true
                            '''
			echo "PyLint Style check"
			sh  ''' . /usr/local/bin/virtualenvwrapper.sh
                                set +e && workon ${BUILD_TAG}
                                mkdir -p reports
                                python3 -m pylint --exit-zero --rcfile=.pylintrc projectq > reports/pylint.log
                            '''				
		    }
		    post {
			always {
			    recordIssues enabledForFailure: true, tools: [pep8(pattern: 'reports/pycodestyle.log'), pyLint(pattern: 'reports/pylint.log')]
			}
		    }
		}

		stage('Unit tests') {
		    steps {
			sh  ''' . /usr/local/bin/virtualenvwrapper.sh
                        set +e && workon ${BUILD_TAG}
                        python3 -m pytest --verbose --junit-xml reports/unit_tests.xml
                    '''
		    }
		    post {
			always {
			    // Archive unit tests for the future
			    junit (allowEmptyResults: true,
				   testResults: 'reports/unit_tests.xml')
			}
		    }
		}		
	    }
	}

        // stage('Integration tests') {
        //     steps {
        //         sh  ''' . /usr/local/bin/virtualenvwrapper.sh
        //                 workon ${BUILD_TAG}
        //                 behave -f=formatters.cucumber_json:PrettyCucumberJSONFormatter -o ./reports/integration.json
        //             '''
        //     }
        //     post {
        //         always {
        //             cucumber (fileIncludePattern: '**/*.json',
        //                       jsonReportDirectory: './reports/',
        //                       parallelTesting: true,
        //                       sortingMethod: 'ALPHABETICAL')
        //         }
        //     }
        // }
	
        // stage('Build package') {
        //     when {
        //         expression {
        //             currentBuild.result == null || currentBuild.result == 'SUCCESS'
        //         }
        //     }
        //     steps {
        //         sh  ''' . /usr/local/bin/virtualenvwrapper.sh
        //                 set +e && workon ${BUILD_TAG}
        //                 python setup.py bdist_wheel  
        //             '''
        //     }
        //     post {
        //         always {
        //             // Archive unit tests for the future
        //             archiveArtifacts (allowEmptyArchive: true,
        //                               artifacts: 'dist/*whl',
        //                               fingerprint: true)
        //         }
        //     }
        // }
	
    }

    post {
        always {
            sh ''' . /usr/local/bin/virtualenvwrapper.sh
                   rmvirtualenv ${BUILD_TAG}'''
        }
    }
}
