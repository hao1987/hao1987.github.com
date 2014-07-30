---
layout: post
title: "PHPUnit Puzzle"
description: ""
category: lessons
tags: [phpunit, tutorial]
---
{% include JB/setup %}

Since learning PHPUnit, I realized how bad my codes are written, didn't take testing into account and after thinking for a little bit, maybe writing test codes are more complicated than writing features, no wonder why we need a testing programmer. 

Found out [this awesome tutorial](https://jtreminio.com/2013/03/unit-testing-tutorial-introduction-to-phpunit/) but better to refer [the official link](http://phpunit.de/manual/current/en/installation.html), it's really dry though.

### Do I really need to insert testing data?
That's the first question came up into my mind, the answer is Yes, at least the fixture data are inserted before your tests but they are or should be cleaned up after tests. Details are listed in the offical sites, getConnection() is supposed to be implemented to get your db connection and getDataSet() is to setup fixtures for you before each test.

Setting up fixture means to define your database's initial state, which is represented by DataSet and DataTable, and I only use File-Based DataSets and DataTables. 

### How to test my featured db functions?
So the idea is:
1. use PHPUnit's db instance to setup fixture
2. create my own db instance, for later calling my feature methods on fixture data
3. assert expectations

### Time to go to a code tour
We implement getConnection() below to fetch our db instance 

~~~php
<?php
abstract class PHPUnit_Extensions_Database_TestCase extends PHPUnit_Extensions_Database_TestCase
{
    static private $pdo = null;

    private $conn = null;

    final public function getConnection()
    {
        if ($this->conn === null) {
        	try {
	            if (self::$pdo == null) {
	                self::$pdo = new PDO( 'mysql:host='.DB_SERVER.';dbname='.DB_DATABASE, DB_USER, DB_PASSWORD );
	            }
	            $this->conn = $this->createDefaultDBConnection(self::$pdo, DB_DATABASE);
        	} catch (PDOException $e) {
        		print_r($e->getMessage());
        	}
        }
        return $this->conn;
    }
}
?>
~~~

then to setup fixture before each test, suppose we have a user table:

~~~php
<?php
function getDataSet()
{
	return new PHPUnit_Extensions_Database_DataSet_YamlDataSet(
		dirname(__FILE__) . '/user.yml'
	);
}
?>
~~~

to use our own db class methods, we have to instantiate it($mysql_obj) somewhere and I put it in setUpBeforeClass(). This instance may vary according to your own requirements, for mine, it has an pdo attribute, so what I do next it to call getConnection() to assign it to that attribute. All in all, the actual db instance I use is instantiated by PHPUnit but this($mysql_obj) can call my featured methods:

~~~php
<?php
class DB_Test extends PHPUnit_Extensions_Database_TestCase
{
	private static $mysql_obj;
	
	public static function setUpBeforeClass()
    {
    	self::$mysql_obj = new mysqldb(USER, PASSWORD, DATABASE, SERVER);
    }
	
	public function setUp() {
		$conn = $this->getConnection();
		$pdo = $conn->getConnection();
		self::$mysql_obj->pdo = $pdo; 
		parent::setUp();
	}
	//...real tests begins...
}
?>
~~~

And of course, we should do the cleaning after each test and close the connection after all:

~~~php
<?php
public function tearDown()
{
	$allTables = $this->getDataSet()->getTableNames();
	foreach ($allTables as $table) {
		// truncate table
	}
	$this->getConnection()->close();
	
	parent::tearDown();
}

public static function tearDownAfterClass()
{
	self::$mysqldb->close(); //our own close() if we have
}
?>
~~~

* We certainly need [DbUnit extension](https://github.com/sebastianbergmann/dbunit)
* Data Schema SHOULD BE already existed, phpunit wont help you with that
* getDataSet() is actually called in setUp()
* Still learning, glad to have your comments :)