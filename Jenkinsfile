pipeline {
    agent any
    // In additional to manual runs, trigger somewhere at midnight to
    // give us the max time in a day to get things right.
    //triggers {
	// Nightly @12am, for "snapshot", skip "release" night.
	//cron('0 0 2-31 * *')
	// First of the month @12am, for "release" (also "current").
	//cron('0 0 1 * *')
    //}
    environment {
	// Pin dates and day to beginning of run.
	START_DATE = sh (
	    script: 'date +%Y-%m-%d',
	    returnStdout: true
	).trim()

	START_DAY = sh (
	    script: 'date +%A',
	    returnStdout: true
	).trim()
	// The branch of geneontology/go-site to use.
	TARGET_GO_SITE_BRANCH = 'iteration'
	// The people to call when things go bad. It is a comma-space
	// "separated" string.
	TARGET_ADMIN_EMAILS = 'sjcarbon@lbl.gov'
	// The file bucket(/folder) combination to use.
	TARGET_BUCKET = 'go-data-product-experimental'
	// The URL prefix to use when creating site indices.
	TARGET_INDEXER_PREFIX = 'http://experimental.geneontology.io'
	// Information for the OSF.io archive.
	OSFIO_USER = 'osf.io@genkisugi.net'
	OSFIO_PROJECT = '6v3gx'
    }
    options{
	timestamps()
    }
    stages {
	// Very first: pause for a few minutes to give a chance to
	// cancel and clean the workspace before use.
	stage('Ready and clean') {
	    steps {
		//sleep time: 1, unit: 'MINUTES'
		cleanWs()
	    }
	}
	stage('Initialize') {
	    steps {
		// Start preparing environment.
		parallel(
		    "Report": {
			sh 'env > env.txt'
			sh 'echo $BRANCH_NAME > branch.txt'
			sh 'echo "$BRANCH_NAME"'
			sh 'cat env.txt'
			sh 'cat branch.txt'
			sh 'echo $START_DAY > dow.txt'
			sh 'echo "$START_DAY"'
		    },
		    "Reset base": {
			// Get a mount point ready
			sh 'mkdir -p $WORKSPACE/mnt || true'
			// Ninja in our file credentials from Jenkins.
			withCredentials([file(credentialsId: 'skyhook-private-key', variable: 'SKYHOOK_IDENTITY')]) {
			    // Try and ssh fuse skyhook onto our local system.
			    sh 'sshfs -o StrictHostKeyChecking=no -o IdentitiesOnly=true -o IdentityFile=$SKYHOOK_IDENTITY -o idmap=user skyhook@skyhook.berkeleybop.org:/home/skyhook $WORKSPACE/mnt/'
			}
			// Remove anything we might have left around from
			// times past.
			sh 'rm -r -f $WORKSPACE/mnt/$BRANCH_NAME || true'
			// Rebuild directory structure.
			sh 'mkdir -p $WORKSPACE/mnt/$BRANCH_NAME/bin || true'
			// WARNING/BUG: needed for arachne to run at
			// this point.
			sh 'mkdir -p $WORKSPACE/mnt/$BRANCH_NAME/lib || true'
			sh 'mkdir -p $WORKSPACE/mnt/$BRANCH_NAME/products || true'
			sh 'mkdir -p $WORKSPACE/mnt/$BRANCH_NAME/products/ttl || true'
			sh 'mkdir -p $WORKSPACE/mnt/$BRANCH_NAME/products/blazegraph || true'
			sh 'mkdir -p $WORKSPACE/mnt/$BRANCH_NAME/products/annotations || true'
			sh 'mkdir -p $WORKSPACE/mnt/$BRANCH_NAME/products/pages || true'
			sh 'mkdir -p $WORKSPACE/mnt/$BRANCH_NAME/products/solr || true'
			sh 'mkdir -p $WORKSPACE/mnt/$BRANCH_NAME/metadata || true'
			sh 'mkdir -p $WORKSPACE/mnt/$BRANCH_NAME/annotations || true'
			sh 'mkdir -p $WORKSPACE/mnt/$BRANCH_NAME/ontology || true'
			sh 'mkdir -p $WORKSPACE/mnt/$BRANCH_NAME/reports || true'
			// Tag the top to let the world know I was at least
			// here.
			sh 'echo "TODO: Note software versions." > $WORKSPACE/mnt/$BRANCH_NAME/manifest.txt'
			sh 'date >> $WORKSPACE/mnt/$BRANCH_NAME/manifest.txt'
			// TODO: This should be wrapped in exception
			// handling. In fact, this whole thing should be.
			sh 'fusermount -u $WORKSPACE/mnt/'
		    }
		)
	    }
	}
        stage('Docker agent test') {
            agent {
                docker {
		    image 'geneontology/golr-autoindex:2018-03-12T180727'
		    // Unsure why Jenkins Docker agent default is to
		    // override with random IDs, but reset to original
		    // root.
		    args '-u root:root -v /tmp/srv-solr-data-exp-01:/srv/solr/data'
		}
            }
            steps {
                sh 'ls /srv'
                sh 'ls /tmp'
		sh 'bash /tmp/run-indexer.sh'
            }
        }
    }
}
