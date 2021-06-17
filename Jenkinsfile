
import groovy.json.JsonSlurper
import groovy.json.JsonOutput; 

def CONFIGDETAILS 
String MISSINGQC = ""
def INSTANCEARN = ""
def TRAGETINSTANCEARN = ""
String PRIMARYQUEUES = ""
String TARGETQUEUES = ""
String PRIMARYRPS = ""
String TARGETRPS = ""

pipeline {
    agent any
    stages {
        stage('git repo & clean') {
            steps {
                script{
                   try{
                      sh(script: "rm -r ac-routing-profiles-syncronization", returnStdout: true)    
                   }catch (Exception e) {
                       echo 'Exception occurred: ' + e.toString()
                   }                   
                   sh(script: "git clone https://github.com/ramprasadsv/ac-routing-profiles-syncronization.git", returnStdout: true)
                   sh(script: "ls -ltr", returnStatus: true)
                   CONFIGDETAILS = sh(script: 'cat parameters.json', returnStdout: true).trim()
                   def config = jsonParse(CONFIGDETAILS)
                   INSTANCEARN = config.primaryInstance
                   TRAGETINSTANCEARN = config.targetInstance
                }
            }
        }
        
        stage('List all Resources') {
            steps {
                echo "List all Resources in both instance "
                withAWS(credentials: '71b568ab-3ca8-4178-b03f-c112f0fd5030', region: 'us-east-1') {
                    script {
                       
                        PRIMARYQUEUES =  sh(script: "aws connect list-queues --instance-id ${INSTANCEARN} --queue-types STANDARD", returnStdout: true).trim()
                        echo PRIMARYQUEUES
                        TARGETQUEUES =  sh(script: "aws connect list-queues --instance-id ${TRAGETINSTANCEARN} --queue-types STANDARD", returnStdout: true).trim()
                        echo TARGETQUEUES
                      
                        PRIMARYRPS =  sh(script: "aws connect list-routing-profiles --instance-id ${INSTANCEARN}", returnStdout: true).trim()
                        echo PRIMARYRPS
                        TARGETRPS =  sh(script: "aws connect list-routing-profiles --instance-id ${TRAGETINSTANCEARN}", returnStdout: true).trim()
                        echo TARGETRPS
                       
                    }
                }
            }
        }
        
        stage('Find missing routing profiles') {
            steps {
                script {
                    echo "Find missing routing profiles in the target instance"
                    def pl = jsonParse(PRIMARYRPS)
                    def tl = jsonParse(TARGETRPS)
                    int listSize = pl.RoutingProfileSummaryList.size() 
                    println "Primary list size $listSize"
                    for(int i = 0; i < listSize; i++){
                        def obj = pl.RoutingProfileSummaryList[i]
                        String qcName = obj.Name
                        String qcId = obj.Id
                        boolean qcFound = checkList(qcName, tl)
                        if(qcFound == false) {
                            println "Missing Name : $qcName Id : $qcId"                                                              
                            MISSINGQC = MISSINGQC.concat(qcId).concat(",")                                
                        }
                    }
                }
                echo "Missing list in the target instance -> ${MISSINGQC}"
            }
        }
        
        stage('Create the missing routing profiles') {
            steps {
                echo "Create the missing queues in the target instance "                
                withAWS(credentials: '71b568ab-3ca8-4178-b03f-c112f0fd5030', region: 'us-east-1') {   
                    script {
                        if(MISSINGQC.length() > 1 ){
                            def qcList = MISSINGQC.split(",")
                            for(int i = 0; i < qcList.size(); i++){
                                String qcId = qcList[i]
                                if(qcId.length() > 2){
                                    def di =  sh(script: "aws connect describe-routing-profile --instance-id ${INSTANCEARN} --routing-profile-id ${qcId}", returnStdout: true).trim()
                                    echo di
                                    def dq =  sh(script: "aws connect list-routing-profile-queues --instance-id ${INSTANCEARN} --routing-profile-id ${qcId}", returnStdout: true).trim()
                                    echo dq
                                    def qc = jsonParse(di)
                                    def rpq = jsonParse(dq)
                                    String qcName = "\'" + qc.RoutingProfile.Name + "\'"
                                    String qcDesc = "\'" + qc.RoutingProfile.Description + "\'"
                                    String chatcc = checkConncurrency(qc.RoutingProfile.MediaConcurrencies, "CHAT")
                                    String voicecc = checkConncurrency(qc.RoutingProfile.MediaConcurrencies, "VOICE")
                                    String taskscc = checkConncurrency(qc.RoutingProfile.MediaConcurrencies, "TASK")
                                    String obQueue = "--default-outbound-queue-id "
                                    obQueue = obQueue + getQueueId (PRIMARYQUEUES, qc.RoutingProfile.DefaultOutboundQueueId, TARGETQUEUES)
                                    echo obQueue
                                    def mc = "[{\"Channel\":\"VOICE\",\"Concurrency\":" + voicecc + "},"                
                                    mc = mc + "{\"Channel\":\"CHAT\",\"Concurrency\":" + chatcc + "},"                
                                    mc = mc + "{\"Channel\":\"TASK\",\"Concurrency\":" + taskscc + "}]"                                                  
                                    echo mc
                                    String json = toJSON(mc)
                                    echo json
                                    mc = "--media-concurrencies " + json
                                    def rpQueueList = getRPQueueList(rpq, PRIMARYQUEUES, TARGETQUEUES)
                                    json = toJSON(rpQueueList)
									echo json
                                    rpQueueList = "--queue-configs " + json     
									echo rpQueueList
                                    rpq = null
                                    qc = null
                                    def cq =  sh(script: "aws connect create-routing-profile --instance-id ${TRAGETINSTANCEARN} --name ${qcName} --description ${qcDesc} ${obQueue} ${mc} ${rpQueueList}  " , returnStdout: true).trim()
                                    echo cq
                               }
                            }
                        }
                    }                
                }
            }
        } 

         stage('queue sync complete') {
            steps {
                echo "completed sycnronizing both instances"                
                withAWS(credentials: '71b568ab-3ca8-4178-b03f-c112f0fd5030', region: 'us-east-1') {   
                    echo "completed sycnronizing both instances"                
                }
            } 
         }
        
     }
}

def getRPQueueList(def qList, def pq, def tq) {
    String ql = "["
    for(int i=0; i < qList.RoutingProfileQueueConfigSummaryList.size(); i++) {
        def obj = qList.RoutingProfileQueueConfigSummaryList[i]
        def q = getQueueIdByName(pq, obj.QueueName, tq)
        String s = '{\"QueueReference\":{\"Channel\":\"' + obj.Channel + '\",\"QueueId\":\"' + q + '\"},\"Priority\":' + obj.Priority + ',\"Delay\":' + obj.Delay + '},'
        ql = ql + s                
    }
    ql = ql.substring(0, ql.length() - 1) 
    ql = ql + "]"
    return ql
}

def checkConncurrency(def mc, def channel ) {
    String cc = "0"
    for (int i = 0; i < mc.size(); i ++) {
        def obj = mc[i]
        if(obj.Channel.equals(channel)) {
            cc = obj.Concurrency
            break
        }
    }
    return cc
}

@NonCPS
def jsonParse(def json) {
    new groovy.json.JsonSlurper().parseText(json)
}

def toJSON(def json) {
    new groovy.json.JsonOutput().toJson(json)
}

def checkList(qcName, tl) {
    boolean qcFound = false
    for(int i = 0; i < tl.RoutingProfileSummaryList.size(); i++){
        def obj2 = tl.RoutingProfileSummaryList[i]
        String qcName2 = obj2.Name
        if(qcName2.equals(qcName)) {
            qcFound = true
            break
        }
    }
    return qcFound
}


def getQueueId (primary, searchId, target) {
    def pl = jsonParse(primary)
    def tl = jsonParse(target)
    String fName = ""
    String rId = ""
    echo "Find for Id : ${searchId}"       
    for(int i = 0; i < pl.QueueSummaryList.size(); i++){
        def obj = pl.QueueSummaryList[i]    
        if (obj.Id.equals(searchId)) {
            fName = obj.Name
            println "Found name : $fName"
            break
        }
    }
    
    echo "Find for name : ${fName}"       
    for(int i = 0; i < tl.QueueSummaryList.size(); i++){
        def obj = tl.QueueSummaryList[i]    
        if (obj.Name.equals(fName)) {
            rId = obj.Id
            println "Found id : $rId"
            break
        }
    }
    
    return rId
    
}


def getQueueIdByName (primary, name, target) {
    def pl = jsonParse(primary)
    def tl = jsonParse(target)
    String fName = ""
    String rId = ""
    echo "Find for name : ${name}"       
    for(int i = 0; i < tl.QueueSummaryList.size(); i++){
        def obj = tl.QueueSummaryList[i]    
        if (obj.Name.equals(name)) {
            fName = obj.Name
            rId = obj.Id
            println "Found name : $fName"
            break
        }
    }
    return rId
    
}

