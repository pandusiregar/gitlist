import groovy.transform.Field
import hudson.AbortException
import hudson.scm.SubversionSCM
import org.tmatesoft.svn.core.SVNDepth
import java.util.concurrent.ConcurrentHashMap

@Field int slaves = 4;
@Field int threadsPerSlave = 2;
@Field boolean coverageBuild = false

if (env.JOB_NAME.contains("coverage")) {
    coverageBuild = true
    env.DOCOVERAGE = 'true'
    println("Doing coverage build")
    threadsPerSlave = 1
}

createStatusVarTable name: "Suites", columns: [
        [name: 'Time', type: "INTERVAL"],
        [name: 'Status', type: "OBJECT"],
        [name: 'Order', type: "NUMBER"]
]

stage name: "Begin", concurrency: 1
properties([
        [$class: 'RequiredResourcesProperty', resourceNames: 'valex-unit'],
        [
                $class  : 'BuildDiscarderProperty',
                strategy: [
                        $class               : 'LogRotator',
                        artifactDaysToKeepStr: '',
                        artifactNumToKeepStr : '',
                        daysToKeepStr        : '30',
                        numToKeepStr         : ''
                ]
        ]
])
List<SubversionSCM.ModuleLocation> scmlocations = new ArrayList<>();

for(i in scm.getLocations()) {
    scmlocations.push(new SubversionSCM.ModuleLocation(i.remote, i.credentialsId, i.local, SVNDepth.INFINITY.name, false))
}

def localscm = ([
        $class               : 'SubversionSCM',
        additionalCredentials: scm.getAdditionalCredentials(),
        locations            : scmlocations,
        workspaceUpdater     : [
                $class: 'UpdateWithCleanUpdater'
        ]
]);

timeout(time: 3, unit: 'HOURS') {
        @Field Boolean[] stashed = new Boolean[slaves]
        @Field Integer[] runCount = new Integer[slaves];

        try {
            node('unittests-master') {
                ws(env.MASTER_UNIT_WORKSPACE) {
                    sh "env"
                    def cwd = pwd()
                    if (!cwd || !env.MASTER_UNIT_WORKSPACE || !cwd.equalsIgnoreCase(env.MASTER_UNIT_WORKSPACE)) {
                        error 'Wrong workspace ' + cwd + " != " + env.MASTER_UNIT_WORKSPACE
                    }
                    stage name: "Checkout", concurrency: 1
                    retry(2) {
                        try {
                            checkout changelog: false, poll: false, scm: localscm
                        } catch(Exception e) {
                            deleteDir()
                            throw e
                        }
                    }
                    withEnv(['PATH+COMPOSER=' + tool('composer') + '/bin',
                             'PATH+ANT=' + tool('ant') + '/bin']) {
                        sh "env"
                        settings = "-Ddatabase.host=${env.DATABASE_HOST} -Ddatabase.name=${env.DATABASE_NAME} -Dbaseaddress=http://${env.MASTER_UNIT_HOSTNAME}/ -Dseleniumserver=${env.seleniumserver}"
                        stage name: "Setup", concurrency: 1
                        parallel db: {
                            setStatusVar name: "Clean DB", value: "Running"
                            def start = System.currentTimeMillis()
                            sh "ant -emacs ${settings} -Dcleandb=true cleandb update-database"
                            def end_time = System.currentTimeMillis()
                            setStatusVar name: "Clean DB", value: end_time - start, type: "INTERVAL"
                        }, setup: {
                            setStatusVar name: "Setup", value: "Running"
                            def start = System.currentTimeMillis()
                            sh "ant -emacs ${settings} build-unit"
                            sh "setfacl -R -m mask::rwx . || true"
                            def end_time = System.currentTimeMillis()
                            setStatusVar name: "Setup", value: end_time - start, type: "INTERVAL"
                        }

                        setStatusVar name: "Installer", value: "Running"
                        def start = System.currentTimeMillis()
                        sh "ant -emacs ${settings} installer vagrant"
                        def end_time = System.currentTimeMillis()
                        setStatusVar name: "Installer", value: end_time - start, type: "INTERVAL"
                    }

                    stage name: 'Test', concurrency: 1

                    loadTasks()
                }
            }


            def tasks = [:]
            def y = 0;
            for (i = 0; i < slaves; i++) {
                tasks['tests' + i] = {
                    def i = y++
                    runCount[i] = 0;
                    node('unittests' + i) {
                        ws(env.SLAVE_UNIT_WORKSPACE) {
                            def cwd = pwd()
                            if (!cwd || !cwd.equalsIgnoreCase(env.SLAVE_UNIT_WORKSPACE)) {
                                error 'Wrong workspace ' + cwd + " != " + env.SLAVE_UNIT_WORKSPACE
                            }
                            retry(2) {
                                try {
                                    checkout changelog: false, poll: false, scm: localscm
                                } catch(Exception e) {
                                    deleteDir()
                                    throw e
                                }
                            }
                            withEnv(['PATH+COMPOSER=' + tool('composer') + '/bin',
                                     'PATH+ANT=' + tool('ant') + '/bin']) {
                                sh "ant -emacs ${settings} build-unit"
                                sh "setfacl -R -m mask::rwx . || true"
                                def subtasks = [:]
                                for (def x = 0; x < threadsPerSlave; x++) {
                                    subtasks['tests' + i + '.' + x] = {
                                        while (runNextTask()) {
                                            runCount[i]++;
                                        }
                                    }
                                }
                                try {
                                    parallel subtasks
                                } finally {
                                    try {
                                        if (runCount[i] > 0) {
                                            if(coverageBuild) {
                                                try {
                                                    sh "php -derror_log='' -dmemory_limit=10024M tests/unit/generate-coverage-report.php"
                                                    sh "sed -i'' -e \"s%`pwd`/%%\" tests/unit/build/logs/*combined.serialized"
                                                } catch (Exception e) {
                                                    println("Coverage failed " + e.toString())
                                                }
                                            }
                                            stash includes: 'tests/unit/build/logs/*-junit.xml,screenshots/**,tests/unit/build/logs/*combined.serialized', name: 'results' + i
                                            stashed[i] = true
                                        } else {
                                            stashed[i] = false;
                                        }
                                    } catch (AbortException e) {
                                        stashed[i] = false;
                                        println "Stash failed for branch " + i + " " + e
                                    }
                                }
                            }
                        }
                    }
                }
            }

            parallel tasks
        } catch(Exception e) {
            println e
            currentBuild.result = 'FAILURE'
        } finally {
            setStatusVar name: "Remaining Suites", value: null
            setStatusVar name: "Running Suites", value: null
            setStatusVar name: "Complete Suites", value: null
            String[] runs = new String[runCount.length];
            for(def i = 0; i < runCount.length; i++) {
                runs[i] = "Slave " + i + " ran " + runCount[i] + " suites";
            }
            println(runs.join("\n"))
            node('master') {
                ws(env.MASTER_UNIT_WORKSPACE) {
                    if (!pwd().equalsIgnoreCase(env.MASTER_UNIT_WORKSPACE)) {
                        error 'Wrong workspace ' + pwd() + " != " + env.MASTER_UNIT_WORKSPACE
                    }
                    stage name: "Report", concurrency: 1
                    for (def i = 0; i < stashed.length; i++) {
                        if (stashed[i] != null && stashed[i]) {
                            try {
                                unstash 'results' + i
                            } catch (Exception e) {
                                println("Failed to unstash " + i + " " + e.toString())
                            }
                        }
                    }
                    dumpResults()
                    sh "php -derror_log='' tests/recordPipelineResults.php"
                    archive 'screenshots/**,testTimings,tasks.list'
                    step([
                            $class                 : 'WarningsPublisher',
                            canComputeNew          : false,
                            canResolveRelativePaths: false,
                            consoleParsers         : [
                                    [parserName: 'Ant Exitcode'],
                                    [parserName: 'PHP Fatal Error'],
                                    [parserName: 'PHP Runtime'],
                                    [parserName: 'PHP Syntax'],
                                    [parserName: 'PHPUnit Coverage Errors']
                            ],
                            defaultEncoding        : '',
                            excludePattern         : '',
                            healthy                : '',
                            includePattern         : '',
                            messagesPattern        : '',
                            unHealthy              : ''
                    ])
                    step([
                            $class        : 'XUnitBuilder',
                            testTimeMargin: '3000',
                            thresholdMode : 1,
                            thresholds    : [
                                    [
                                            $class              : 'FailedThreshold',
                                            failureNewThreshold : '',
                                            failureThreshold    : '',
                                            unstableNewThreshold: '',
                                            unstableThreshold   : '0'
                                    ], [
                                            $class              : 'SkippedThreshold',
                                            failureNewThreshold : '',
                                            failureThreshold    : '',
                                            unstableNewThreshold: '',
                                            unstableThreshold   : '0']
                            ],
                            tools         : [
                                    [
                                            $class               : 'JUnitType',
                                            deleteOutputFiles    : true,
                                            failIfNotNew         : true,
                                            pattern              : 'tests/unit/build/logs/*-junit.xml',
                                            skipNoTestFiles      : true,
                                            stopProcessingIfError: false
                                    ]
                            ]
                    ])

                    recordUnitTestsToDB()

                    if(coverageBuild) {
                        try {
                            stage name: "Coverage"
                            sh "php -derror_log='' tests/unit/merge-junit.php\n" +
                                    "sed -i'' -e \"s%`pwd`/%%\" tests/unit/build/logs/junit.xml"

                            withEnv(['PATH+COMPOSER=' + tool('composer') + '/bin',
                                     'PATH+ANT=' + tool('ant') + '/bin']) {
                                sh "ant -emacs ${settings} coverage"
                            }
                            archive 'tests/unit/build/logs/clover.xml'
                            step([$class: 'CloverPublisher', cloverReportDir: 'tests/unit/build/logs/', cloverReportFileName: 'clover.xml'])

                            stage name: "Sonar"
                            withEnv(['PATH+SONAR=' + tool(name: 'Sonar 2.4', type: 'hudson.plugins.sonar.SonarRunnerInstallation') + "/bin/"]) {
                                sh 'sonar-runner xxxxxxxxxxxxxxxxxx -Dsonar.projectBaseDir=' + pwd() + ' -Dsonar.build-stability.url=Jenkins:' + env.JOB_URL
                            }
                        } catch (err) {
                            echo "Caught: ${err}"
                            currentBuild.result = 'FAILURE'
                        }
                    }

                    step([$class: 'Mailer', notifyEveryUnstableBuild: true, recipients: unitTestBreakersEMails() + " " + commiters(), sendToIndividuals: true])
                }
            }
        }
    }

@Field taskResults = new ConcurrentHashMap<String, Long>()
@Field Stack taskList = new Stack();

def loadTasks() {
    taskList.addAll(readFile('tasks.list').split('\n'))
    setStatusVar name: "Remaining Suites", value: taskList.size(), type: "NUMBER"
    setStatusVar name: "Total Suites", value: taskList.size(), type: "NUMBER"
    setStatusVar name: "Running Suites", type: 'NUMBER', value: 0
    setStatusVar name: "Complete Suites", type: 'NUMBER', value: 0
    for(def i = 0; i < taskList.size(); i++) {
        setStatusVar table: "Suites", name: taskList.get(i), column: 1, value: "Queued"
        setStatusVar table: "Suites", name: taskList.get(i), column: 2, value: taskList.size() - i, type: "NUMBER"
    }
}

def runNextTask() {
    try {
        if(currentBuild.result == "ABORTED") {
            return false;
        }
        def task = taskList.pop();
        def start = System.currentTimeMillis()
        setStatusVar name: "Remaining Suites", value: taskList.size(), type: "NUMBER"
        setStatusVar table: "Suites", name: task, column: 1, value: "Running"
        try {
            incStatusVar name: "Running Suites", type: 'NUMBER'
            sh "ant -emacs -file tests/unit/build.xml " + task
            setStatusVar table: "Suites", name: task, column: 1, value: "Complete"
        } catch(InterruptedException e) {
            setStatusVar table: "Suites", name: task, column: 1, value: "Aborted"
            throw e
        } catch(Exception e) {
            appendStatusVar name: "Failed", value: task
            setStatusVar table: "Suites", name: task, column: 1, value: "Failed"
            println(e)
            currentBuild.result = 'FAILURE'
        } finally {
            decStatusVar name: "Running Suites", type: 'NUMBER'
            incStatusVar name: "Complete Suites", type: 'NUMBER'
        }
        def end_time = System.currentTimeMillis();
        taskResults.put(task, end_time - start);
        setStatusVar table: "Suites", name: task, column: 0, value: end_time - start, type: "INTERVAL"
        return true
    } catch (EmptyStackException e) {
        return false
    }
}

def dumpResults() {
    def results = ''
    for (i in taskResults) {
        results += i.key + "=" + i.value + "\n"
    }
    println results
    writeFile file: 'testTimings', text: results
}
