# cluster
Hints for creating an node.js cluster for an Angular application

In this lab you will create a node cluster for an Angular application that will display the process ids for the node process that is handling a http REST service. The following steps may help you in implementing the lab.
1. First, create a express application

 ```
express --ejs cluster
 ```
2. This will create the full framework. Edit bin/www to change the port to the one you want, run "npm install" and you should be able to run the application with

 ```
npm start
 ```
3. Now modify the ejs to display the PID of the process handling the HTTP request. Change views/index.ejs to have the line.

 ```html
   <p>Welcome to <%= title %> from <%= pid %></p>
 ```
4. And change the controller in routes/index.js to populate the pid field

  ```js
router.get('/', function(req, res, next) {
  console.log("ID "+process.pid);
  res.render('index', { title: 'Cluster', pid:process.pid });
});
  ```
 
5. You should now be able to test the application using the route http://YOURIP/

6. Now add your REST service to get the PID in routes/index.js and tell the browser not to cache the results

  ```js
router.get('/pid', function(req, res, next) {
  console.log("Service ID "+process.pid);
  res.header("Cache-Control", "no-cache, no-store, must-revalidate");
  res.header("Pragma", "no-cache");
  res.header("Expires", 0);
  res.json({ title: 'Cluster', pid:process.pid });
});
  ```
6. You should be able to test this service at http://YOURIP/pid
Now lets build an angular application to consume this service. First build the view in views/index.ejs and point it to the MainCtrl front end controller in public/javascripts/angularApp.js

  ```html
<html>
<head>
  <title>Cluster</title>
  <script src="//ajax.googleapis.com/ajax/libs/angularjs/1.2.19/angular.min.js"></script>

<script src="/javascripts/angularApp.js"></script>
</head>

<body ng-app="clusterApp" ng-controller="MainCtrl">
    <ul>
    <div ng-repeat="node in cluster">
      <li>{{node.pid}}</li>
    </div>
   </ul>
</body>
</html>
   ```
7. We will need to populate the cluster array in our MainCtrl

  ```js
angular.module('clusterApp', [])
.controller('MainCtrl', [
  '$scope', '$http',
  function($scope, $http){
    $scope.cluster = [{pid:1234}];
  } 
]);   
  ```
 
8. Test the application to make sure it works.
Now add a button to get the PID from the front end controller. For now, we just want to make sure we can update the $scope.cluster array correctly. Add to the views/index.ejs

  ```html
    <form ng-submit="getPIDs()">
      <h3>Test PIDs </h3>
      <button type="submit">TestPIDS</button>
    </form>
  ```
 
9. And implement the getPIDs() function in public/javascripts/angularApp.js

  ```js
    $scope.getPIDs = function() {
      $scope.cluster = [{pid:12},{pid:34}];
    }
  ```
10. You should be able to test this to make sure the view displays these two fictional pids.
Now add another button to access your REST service in views/index.ejs

  ```html
    <form ng-submit="getMyPIDs()">
      <h3>Display PIDs </h3>
      <button type="submit">GETPIDS</button>
    </form>
 ```
 
11. And implement the getMyPIDs function in public/javascripts/angularApp.js

  ```js
    $scope.getMyPIDs = function() {
        $http.get('/pid').success(function(data){
          console.log("getAll");
          console.log(data);
          $scope.cluster.push(data);
        });
    }
  ```
11. You should be able to test this button to make sure it displays the PID of your REST service.
Now modify your front end controller to make 100 http calls and push them all onto the $scope.cluster. You should notice that the all have the same PID.
Now that we have the angular application making a bunch of requests, we need to spawn multiple processes to handle requests to our REST service. Remember that when you run "npm start", npm is really just running the script in "bin/www". So, we should be able to spawn a bunch of workers and have them run "bin/www". Create a cluster.js file to do this.

  ```js
var cluster = require('cluster');
var numCPUs = require('os').cpus().length;

if (cluster.isMaster) {
  console.log("numCPUs "+numCPUs);
  // OK, lets ignore the actual CPUS, so we can see the concurrency
  numCPUs = 4;

  for (var i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on('exit', function(worker, code, signal) {
    console.log('worker ' + worker.process.pid + ' died');
  });
} else {
    console.log("Worker "+process.pid);
    //change this line to Your Node.js app entry point.
    require("./bin/www");
}
  ```
 
12. now run "sudo node cluster.js" and see what happens with your angular app
You should be able to run

  ```
ab -n 100 -v 10 http://YOURIP/pid
  ```
to see that your service is returning different PIDs depending on which process handles your REST service. If you have a machine with multiple processors, these would all execute concurrently.
