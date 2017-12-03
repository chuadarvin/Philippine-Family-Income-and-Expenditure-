Important if you want to restart again 
stop all containers: 
docker kill $(docker ps -q)
remove all containers. 
docker rm $(docker ps -a -q)

docker network create testmongo

// I. SETUP DOCKER
// 1. Create your project folder.


// 2. Create a "docker-compose.yml" file in your project folder.

// 3. Copy the contents of the "docker-compose.yml template" to the newly created "docker-compose.yml"

// 4. Start your services
docker-compose up

// 5. Login to "mongosetup"
docker-compose run mongosetup sh

// 6. Login to "mongo1"
mongo --host mongo1:27017 book

// II. SETUP REPLICA SET
// 1. Setup the configuration for the replica set
var cfg = {
	"_id": "book",
	"version": 1,
	"members": [
		{
			"_id": 0,
			"host": "mongo1:27017",
			"priority": 3
		},
		{
			"_id": 1,
			"host": "mongo2:27017",
			"priority": 1
		},
		{
			"_id": 2,
			"host": "mongo3:27017",
			"priority": 2
		}
        
	]
}

// 2. Initiate the replica set using the configuration
rs.initiate( cfg );

exit


// 4. Login to the secondary node i.e. 'mongo2'
mongo --host mongo2:27017 book

// 5. Everytime you switch a node, execute this command.
db.setSlaveOk()

// 6. Check collections in this node
show collections;

exit



// back to main
mongo --host mongo1:27017 book

//
db.setSlaveOk()


// move file to directory and then mongo import
exit

docker-compose run mongosetup sh
// edited docker compose file to include volume for mongosetup and moved the csv file into the resulting folder 

mongoimport --host book/mongo1:27017,mongo2:27017,mongo3:27017 --db phdata --collection phincomeexpenditure --type csv --headerline --file /data/db/Family\ Income\ and\ Expenditure.csv

 // 
mongo --host mongo1:27017 book
// 
show databases
//
use phdata

db.phincomeexpenditure.find().limit(1).pretty()

//
exit

docker ps
docker exec -it <containernameofprimary> bash
mongo 
use admin
db.adminCommand({shutdown : 1, force : true});


// 4. Login to the secondary node i.e. 'mongo2'
mongo --host mongo2:27017 book

// 5. Everytime you switch a node, execute this command.
db.setSlaveOk()

// 6. Check collections in this node
use phdata
db.setSlaveOk()
show collections;


// 7. Check if the document recorded from 'mongo1' has been replicated to 'mongo2'
db.phincomeexpenditure.find().limit(1).pretty()













// I. SETUP DOCKER
// 1. Create your project folder.
mkdir sharding

// 2. Create a "docker-compose.yml" file in your project folder.

// 3. Copy the contents of the "docker-compose.yml template" to the newly created "docker-compose.yml"

// 4. Start your services
docker-compose up

// 5. Login to "mongosetup"
docker-compose run mongosetup2 sh


mongo --host mongo4:27019



rs.initiate({
   _id: "config",
   configsvr: true,
   members: [
      { _id: 0, host: "mongo4:27019" },
      { _id: 1, host: "mongo5:27019" },
      { _id: 2, host: "mongo6:27019" }
   ]
} )

rs.reconfig({
   _id: "config",
   configsvr: true,
   members: [
      { _id: 0, host: "mongo4:27019" },
      { _id: 1, host: "mongo5:27019" },
      { _id: 2, host: "mongo6:27019" }
   ]
}, { force: true } )

exit

// had some edits on the dockercompose file to include networking 
docker exec -it <containerofmongos> bash
mongo
sh.addShard( "book/mongo2:27017,mongo3:27017" )


// make another replica set
// create another folder 

docker-compose run mongosetup3 sh

mongo --host mongo7:27017

rs.initiate({
   _id: "book2",
   members: [
      { _id: 0, host: "mongo7:27017" },
      { _id: 1, host: "mongo8:27017" },
      { _id: 2, host: "mongo9:27017" }
   ]
} )

// back to mongos
sh.addShard( "book2/mongo7:27017,mongo8:27017" )



sh.enableSharding( "phdata" )

use phdata

//create index
db.phincomeexpenditure.createIndex(
	{ 'Region': 1 },
	{ name: 'region' }
)



//shard the collection
sh.shardCollection( 
	"phdata.phincomeexpenditure", 
	{ "Region": 1 }
)

// to see if everything is okay
db.stats()
db.printShardingStatus()





// IV. EXECUTE MAPREDUCE ON SHARDED COLLECTION
// After sharding a collection, we can run a mapreduce job and shard the output.

// 1. Define map and reduce function to find the number of phone records per prefix.
map = function() {
	emit({
		prefix: this.Region
	}, {
		count: 1
	});
}

reduce = function(key, values) {
	var total = 0;
	for( var i = 0; i < values.length; i++ ) {
		total += values[i].count;
	}
	return { count: total };
}

// 2. Execute the map reduce job.
results = db.runCommand({
	mapReduce: 'phincomeexpenditure',
	map: map,
	reduce: reduce,
	out: { 
		replace: 'phincomeexpenditure.report',
		sharded: true
	}
});

db.phincomeexpenditure.report.find()

