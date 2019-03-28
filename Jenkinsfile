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
		step([$class: 'LastChangesPublisher',
		      since:'PREVIOUS_REVISION',
		      format: 'SIDE',
		      matchWordsThreshold: '0.25',
		      matching: 'NONE',
		      matchingMaxComparisons: '2500',
		      showFiles: true,
		      synchronisedScroll: true])
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
                        pip$PY install pylint pycodestyle flake8
                        pip$PY install dormouse
                        
                        if [[ "$OSTYPE" == "darwin"* ]]; then
                            export CC=clang
                            export CXX=clang++
                            export LDFLAGS=-L`brew --prefix llvm`/lib
                            # for i in {10..1}; do
                            #     if [[ `command -v gcc-${i}` ]]; then
                            #         export CC=gcc-${i}
                            #         export CXX=g++-${i}
                            #         break
                            #     fi
                            # done
                        else
                            export CC=gcc
                            export CXX=g++
                        fi
  
                        pip$PY install revkit --no-binary=revkit --ignore-installed
                        pip$PY install -e .
                    '''
            }
        }

	stage('Quality check') {
	    parallel {
		stage('PEP8 - PyCodeStyle') {
		    steps {
			echo "PEP8 Style check"
			sh  ''' . /usr/local/bin/virtualenvwrapper.sh
                                set +e && workon ${BUILD_TAG}
                                mkdir -p reports
                                python3 -m pycodestyle --exclude '*_test.py' projectq > reports/pycodestyle.log || true
                            '''
		    }
		}
		stage('PyLint') {
		    steps {
			echo "PyLint Style check"
			sh  ''' . /usr/local/bin/virtualenvwrapper.sh
                                set +e && workon ${BUILD_TAG}
                                mkdir -p reports
                                python3 -m pylint -j 2 --exit-zero --rcfile=.pylintrc projectq > reports/pylint.log
                            '''
		    }
		}
		stage('Flake8') {
		    steps {
			echo "Flake8 Style check"
			sh  ''' . /usr/local/bin/virtualenvwrapper.sh
                                set +e && workon ${BUILD_TAG}
                                mkdir -p reports
                                python3 -m flake8 --format pylint --exclude '*_test.py' --exit-zero --output-file reports/flake8.log  projectq
                            '''
		    }
		}
		stage('PMD CPD') {
		    steps {
			echo "PMD Copy-paste detector"
			sh  ''' mkdir -p reports
                                find projectq -name '*.py' -and ! -name '*_test.py' -and ! -name '.*' > file_list.txt
                                pmd cpd --language python --format xml --minimum-tokens 100 --failOnViolation false --filelist file_list.txt > reports/cpd.xml
                            '''
		    }
		}
	    }
	    post {
		always {
		    recordIssues enabledForFailure: true, tools: [pep8(pattern: 'reports/pycodestyle.log'),
								  pyLint(pattern: 'reports/pylint.log'),
								  flake8(pattern: 'reports/flake8.log'),
								  cpd(pattern: 'reports/cpd.xml'),
								  taskScanner(excludePattern: '*/*_test.py, */*/*_test.py, */*/*/*_test.py',
									      highTags: 'FIXME',
									      ignoreCase: true,
									      includePattern: '*/*.py, */*/*.py, */*/*/*.py',
									      normalTags: 'TODO')]
		}
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
                        echo '[run]' > .coveragerc
                        echo 'omit = *_test.py' >> .coveragerc
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
