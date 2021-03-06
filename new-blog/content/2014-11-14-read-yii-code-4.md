Title: Yii源码阅读笔记 - Model层实现
Date: 2014-11-14
Author: youngsterxyf
Slug: read-yii-code-4
Tags: PHP, Yii, 笔记, 总结

### 概述

Yii中，对Model层的使用，有两种方式：

1. 通过类CDbConnection和CDbCommand来操作
2. 使用ORM形式：编写model类继承自抽象类CActiveRecord

第1种方式的示例如下：

    :::php
    <?php
    $connection = Yii::app()->db;  // 或者Yii::app()->getComponent('db');
    $queryResult = $connection->createCommand($sql)->queryRow();

第2种方式中编写的model类可能需要实现方法`getDbConnection`、`model`、`tableName`。

在实现上，第2种方式是基于第1种方式的，即第2种方式的抽象程度更高。Yii没有屏蔽第1种方式，这样能让开发者按需选择。
但我个人并不喜欢这样，两种方式同时存在，会导致应用的model实现稍显混乱。

### 分析

Yii框架model层的入口为CDbConnection类，该类有很多public的属性可供配置，如`connectionString`、`username`、`password`等。

根据[Yii源码阅读笔记 - 组件集成](http://youngsterxyf.github.io/2014/11/13/read-yii-code-3/)一文可知，组件初始化时会调用init方法。
类CDbConnection的init类实现如下：

    :::php
    public function init()
    {
        parent::init();
        // 属性autoConnect默认为true
        if($this->autoConnect)
            $this->setActive(true);
    }

其中调用的setActive方法实现如下：

    :::php
    public function setActive($value)
    {
        // 当$value为true，而_active为false（表示数据库连接未打开），则打开数据库连接
        // 当$value为false, 而_active为true（表示数据库连接已打开），则关闭数据库连接
        if($value!=$this->_active)
        {
            if($value)
                $this->open();
            else
                $this->close();
        }
    }

方法open实现如下：

    :::php
    /**
     * Opens DB connection if it is currently not
     * @throws CException if connection fails
     */
    protected function open()
    {
        if($this->_pdo===null)
        {
            // 所以需要配置connectionString
            if(empty($this->connectionString))
                throw new CDbException('CDbConnection.connectionString cannot be empty.');
            try
            {
                Yii::trace('Opening DB connection','system.db.CDbConnection');
                // 基于PDO类建立数据库连接（对于某些数据库不使用PDO）
                $this->_pdo=$this->createPdoInstance();
                // 设置数据库连接的一些属性，如字符编码等
                $this->initConnection($this->_pdo);
                // 标志位设置为已打开
                $this->_active=true;
            }
            catch(PDOException $e)
            {
                // 省略
            }
        }
    }

方法close实现如下：

    :::php
    /**
     * Closes the currently active DB connection.
     * It does nothing if the connection is already closed.
     */
    protected function close()
    {
        Yii::trace('Closing DB connection','system.db.CDbConnection');
        $this->_pdo=null;
        $this->_active=false;
        $this->_schema=null;
    }

open方法中调用的方法createPdoInstance实现如下：

    :::php
    /**
     * Creates the PDO instance.
     * When some functionalities are missing in the pdo driver, we may use
     * an adapter class to provide them.
     * @throws CDbException when failed to open DB connection
     * @return PDO the pdo instance
     */
    protected function createPdoInstance()
    {
        // 属性pdoClass默认为PDO
        $pdoClass=$this->pdoClass;
        if(($pos=strpos($this->connectionString,':'))!==false)
        {
            // 取出数据库驱动类型
            $driver=strtolower(substr($this->connectionString,0,$pos));
            if($driver==='mssql' || $driver==='dblib')
                $pdoClass='CMssqlPdoAdapter';
            elseif($driver==='sqlsrv')
                $pdoClass='CMssqlSqlsrvPdoAdapter';
        }

        if(!class_exists($pdoClass))
            throw new CDbException(Yii::t('yii','CDbConnection is unable to find PDO class "{className}". Make sure PDO is installed correctly.',
                array('{className}'=>$pdoClass)));

        // 实例化类PDO，可能失败
        @$instance=new $pdoClass($this->connectionString,$this->username,$this->password,$this->_attributes);

        if(!$instance)
            throw new CDbException(Yii::t('yii','CDbConnection failed to open the DB connection.'));

        return $instance;
    }

从中可以看出，对于MySQL数据库而言，方法createPdoInstance返回的是一个PDO对象，赋值给CDbConnection对象的_pdo属性。

方法initConnection的实现如下：

    :::php
    /**
     * Initializes the open db connection.
     * This method is invoked right after the db connection is established.
     * The default implementation is to set the charset for MySQL and PostgreSQL database connections.
     * @param PDO $pdo the PDO instance
     */
    protected function initConnection($pdo)
    {
        // 设置数据库连接的一些属性
        $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
        if($this->emulatePrepare!==null && constant('PDO::ATTR_EMULATE_PREPARES'))
            $pdo->setAttribute(PDO::ATTR_EMULATE_PREPARES,$this->emulatePrepare);
        if($this->charset!==null)
        {
            $driver=strtolower($pdo->getAttribute(PDO::ATTR_DRIVER_NAME));
            if(in_array($driver,array('pgsql','mysql','mysqli')))
                $pdo->exec('SET NAMES '.$pdo->quote($this->charset));
        }
        // 执行一些初始化的SQL语句
        // public $initSQLs : array list of SQL statements that should be executed right after the DB connection is established.
        if($this->initSQLs!==null)
        {
            foreach($this->initSQLs as $sql)
                $pdo->exec($sql);
        }
    }

------

由于第2种方式是基于第1中方式实现的，所以我们先看看第1种方式的实现。

类CDbConnection中方法createCommand的实现如下：

    :::php
    public function createCommand($query=null)
    {
        $this->setActive(true);
        return new CDbCommand($this,$query);
    }

其中实例化的类CDbCommand的构造方法实现如下：

    :::php
    /**
     * Constructor.
     * @param CDbConnection $connection the database connection
     * @param mixed $query the DB query to be executed. This can be either
     * a string representing a SQL statement, or an array whose name-value pairs
     * will be used to set the corresponding properties of the created command object.
     *
     * For example, you can pass in either <code>'SELECT * FROM tbl_user'</code>
     * or <code>array('select'=>'*', 'from'=>'tbl_user')</code>. They are equivalent
     * in terms of the final query result.
     *
     * When passing the query as an array, the following properties are commonly set:
     * {@link select}, {@link distinct}, {@link from}, {@link where}, {@link join},
     * {@link group}, {@link having}, {@link order}, {@link limit}, {@link offset} and
     * {@link union}. Please refer to the setter of each of these properties for details
     * about valid property values. This feature has been available since version 1.1.6.
     *
     * Since 1.1.7 it is possible to use a specific mode of data fetching by setting
     * {@link setFetchMode FetchMode}. See {@link http://www.php.net/manual/en/function.PDOStatement-setFetchMode.php}
     * for more details.
     */
    public function __construct(CDbConnection $connection,$query=null)
    {
        $this->_connection=$connection;
        if(is_array($query))
        {
            foreach($query as $name=>$value)
                $this->$name=$value;
        }
        else
            $this->setText($query);
    }

可以看到参数$query并不是必须提供，如果提供，则$query参数值可以是字符串也可以是数组。如果是字符串也就是一个原生的SQL语句（可能有参数需要填充），如果是数组，
则可以为CDbCommand对象的select、distinct、from等属性赋值。

方法setText实现如下：

    :::php
    /**
     * Specifies the SQL statement to be executed.
     * Any previous execution will be terminated or cancel.
     * @param string $value the SQL statement to be executed
     * @return CDbCommand this command instance
     */
    public function setText($value)
    {
        if($this->_connection->tablePrefix!==null && $value!='')
            $this->_text=preg_replace('/{{(.*?)}}/',$this->_connection->tablePrefix.'\1',$value);
        else
            $this->_text=$value;
        // 因为是要新执行一条语句，所以需要重置状态，将属性_statement置为null。
        $this->cancel();
        return $this;
    }

如果调用类CDbConnection的createCommand方法时提供了`$query`参数值，则在得到CDbCommand对象后，即可调用CDbCommand对象的execute方法（对于增、删、改操作）或query、queryAll、queryRow等方法（对于查询操作）来执行数据库操作。

如果调用createCommand时未提供`$query`参数值，则有3种方式来完成数据库操作：

1. 对于普通的数据库查询操作，得到CDbCommand对象后，链式调用方法select、from、where等（之所以能够链式调用，是因为这些方法的最后都返回了对象本身），并且链式调用的最后调用query一类方法来执行数据库操作。

2. 对于会对数据表结构或数据产生变更的操作，得到CDbCommand对象后，可以直接调用方法insert、update、delete、createTable等来执行操作。

3. 可以在得到CDbCommand对象后，调用方法setText来设置待执行的SQL语句，setText方法的$value参数类型应为字符串。如果调用setText方法前$value参数类型是关联数组，则可以先调用方法buildQuery从关联数组生成一个SQL语句字符串，再调用setText方法；或者直接调用setSelect、setFrom、setWhere等方法来设置SQL语句的各个组成部分。最后调用execute方法或query一类方法。可以看出相比传递`$query`参数，这种方式只不过是显式地设置SQL语句。通常这是不推荐的（搞这么麻烦，何必呢？呵呵）。

这样一看，类CDbCommand的实现稍显混乱。本意上额外的第3种方式应该是提供给类CActiveRecord使用的，所以不建议使用。

接下来看看方法execute方法及query一类（仅以方法query、queryAll为例）方法的实现：

    :::php
    /**
     * Executes the SQL statement.
     * This method is meant only for executing non-query SQL statement.
     * No result set will be returned.
     * @param array $params input parameters (name=>value) for the SQL execution. This is an alternative
     * to {@link bindParam} and {@link bindValue}. If you have multiple input parameters, passing
     * them in this way can improve the performance. Note that if you pass parameters in this way,
     * you cannot bind parameters or values using {@link bindParam} or {@link bindValue}, and vice versa.
     * Please also note that all values are treated as strings in this case, if you need them to be handled as
     * their real data types, you have to use {@link bindParam} or {@link bindValue} instead.
     * @return integer number of rows affected by the execution.
     * @throws CDbException execution failed
     */
    public function execute($params=array())
    {
        if($this->_connection->enableParamLogging && ($pars=array_merge($this->_paramLog,$params))!==array())
        {
            $p=array();
            foreach($pars as $name=>$value)
                $p[$name]=$name.'='.var_export($value,true);
            $par='. Bound with ' .implode(', ',$p);
        }
        else
            $par='';
        Yii::trace('Executing SQL: '.$this->getText().$par,'system.db.CDbCommand');
        try
        {
            if($this->_connection->enableProfiling)
                Yii::beginProfile('system.db.CDbCommand.execute('.$this->getText().$par.')','system.db.CDbCommand.execute');

            // 以通过setText设置的SQL语句为参数间接调用PDO对象的prepare方法，并将返回的PDOStatement对象赋值给当前CDbCommand对象的_statement属性。
            $this->prepare();
            if($params===array())
                // 无参执行
                $this->_statement->execute();
            else
                // 带参执行
                $this->_statement->execute($params);
            // 操作影响的数据表行数
            $n=$this->_statement->rowCount();

            if($this->_connection->enableProfiling)
                Yii::endProfile('system.db.CDbCommand.execute('.$this->getText().$par.')','system.db.CDbCommand.execute');

            return $n;
        }
        catch(Exception $e)
        {
            // 省略
        }
    }

    public function query($params=array())
    {
        return $this->queryInternal('',0,$params);
    }
    
    public function queryAll($fetchAssociative=true,$params=array())
    {
        return $this->queryInternal('fetchAll',$fetchAssociative ? $this->_fetchMode : PDO::FETCH_NUM, $params);
    }

可以看到query一类方法都是间接调用方法queryInternal来完成操作的。queryInternal方法实现如下：

    :::php
    private function queryInternal($method,$mode,$params=array())
    {
        $params=array_merge($this->params,$params);

        if($this->_connection->enableParamLogging && ($pars=array_merge($this->_paramLog,$params))!==array())
        {
            $p=array();
            foreach($pars as $name=>$value)
                $p[$name]=$name.'='.var_export($value,true);
            $par='. Bound with '.implode(', ',$p);
        }
        else
            $par='';

        Yii::trace('Querying SQL: '.$this->getText().$par,'system.db.CDbCommand');

        // 先尝试从缓存中读取查询结果
        // 这里涉及CDbConnection类的三个属性queryCachingCount、queryCachingDuration、queryCacheID
        // 另外对于方法query（调用queryInternal时提供的method参数为空字符串）的操作不会缓存
        if($this->_connection->queryCachingCount>0 && $method!==''
                && $this->_connection->queryCachingDuration>0
                && $this->_connection->queryCacheID!==false
                && ($cache=Yii::app()->getComponent($this->_connection->queryCacheID))!==null)
        {
            $this->_connection->queryCachingCount--;
            $cacheKey='yii:dbquery'.$this->_connection->connectionString.':'.$this->_connection->username;
            $cacheKey.=':'.$this->getText().':'.serialize(array_merge($this->_paramLog,$params));
            if(($result=$cache->get($cacheKey))!==false)
            {
                Yii::trace('Query result found in cache','system.db.CDbCommand');
                return $result[0];
            }
        }

        try
        {
            if($this->_connection->enableProfiling)
                Yii::beginProfile('system.db.CDbCommand.query('.$this->getText().$par.')','system.db.CDbCommand.query');

            $this->prepare();
            if($params===array())
                $this->_statement->execute();
            else
                $this->_statement->execute($params);

            // $method对应PDOStatement的结果获取方法，如果未提供$method，自然无法直接通过PDOStatement获取查询结果。
            if($method==='')
                // 这个细看一下
                $result=new CDbDataReader($this);
            else
            {
                $mode=(array)$mode;
                call_user_func_array(array($this->_statement, 'setFetchMode'), $mode);
                // 调用PDOStatement对应的方法
                $result=$this->_statement->$method();
                $this->_statement->closeCursor();
            }

            if($this->_connection->enableProfiling)
                Yii::endProfile('system.db.CDbCommand.query('.$this->getText().$par.')','system.db.CDbCommand.query');
            
            // 如果设置了$cache和$cacheKey
            // 将查询结果存入缓存
            if(isset($cache,$cacheKey))
                $cache->set($cacheKey, array($result), $this->_connection->queryCachingDuration, $this->_connection->queryCachingDependency);

            return $result;
        }
        catch(Exception $e)
        {
            // 省略
        }
    }

可以看到query方法调用queryInternal时，结果是通过CDbDataReader对象来获取的。类CDbDataReader的构造方法实现如下：

    :::php
    public function __construct(CDbCommand $command)
    {
        $this->_statement=$command->getPdoStatement();
        $this->_statement->setFetchMode(PDO::FETCH_ASSOC);
    }

得到CDbDataReader对象，即可通过它的read一类方法**迭代**获取查询结果。这些方法实际上是调用PDOStatement的fetch一类方法来获取结果的。

------

再来看看Yii框架Model层的第2种使用方式。

CActiveRecord的用法是，对于数据库的每个数据表，创建一个model类，如UserModel，这个类继承自CActiveRecord类。model类名可以和数据表名一致，也可以不一致，如果不一致，则需要重写CActiveRecord类的tableName方法，标明该model类对应的数据表名。方法tableName默认的实现如下：

    :::php
    /**
    * Returns the name of the associated database table.
    * By default this method returns the class name as the table name.
    * You may override this method if the table is not named after this convention.
    * @return string the table name
    */
    public function tableName()
    {
        return get_class($this);
    }

假设UserModel类对应的数据表名为User，则应如下重写tableName：

    :::php
    public function tableName()
    {
        return 'User';
    }

使用CActiveRecord方式，若想向数据表中插入一条新纪录，则需要实例化当前model类。以UserModel类为例，若想向User表中插入一条新纪录，则需要先实例化UserModel，得到一个对象，然后对该对象的属性进行赋值指明对应数据表新记录每个字段的值。对象的属性名即数据表的字段名，赋值完毕，调用save或insert即向数据表中存入该新纪录。

CActiveRecord类的构造方法如下所示：

    :::php
    public function __construct($scenario='insert')
    {
        if($scenario===null) // internally used by populateRecord() and model()
            return;

        $this->setScenario($scenario);
        $this->setIsNewRecord(true);
        $this->_attributes=$this->getMetaData()->attributeDefaults;

        $this->init();

        $this->attachBehaviors($this->behaviors());
        $this->afterConstruct();
    }

$scenario='insert'，表示新实例化的对象处于待插入数据表的状态，在调用save等方法时，会检测该状态。在新实例化的对象插入到数据表后，该状态立即会变为"update"，表示之后对该对象的数据库写入操作，属于update操作。

CActiveRecord类的save方法的实现如下：

    :::php
    public function save($runValidation=true,$attributes=null)
    {
        // 若需要则对某些属性做校验
        if(!$runValidation || $this->validate($attributes))
            // 如果是新纪录，则插入，否则更新
            return $this->getIsNewRecord() ? $this->insert($attributes) : $this->update($attributes);
        else
            return false;
    }

而insert、update方法的实现如下所示：

    :::php
    public function insert($attributes=null)
    {
        if(!$this->getIsNewRecord())
            throw new CDbException(Yii::t('yii','The active record cannot be inserted to database because it is not new.'));
        if($this->beforeSave())
        {
            Yii::trace(get_class($this).'.insert()','system.db.ar.CActiveRecord');
            // ...
            $builder=$this->getCommandBuilder();
            // ...
            $table=$this->getMetaData()->tableSchema;
            $command=$builder->createInsertCommand($table,$this->getAttributes($attributes));
            if($command->execute())
            {
                $primaryKey=$table->primaryKey;
                if($table->sequenceName!==null)
                {
                    if(is_string($primaryKey) && $this->$primaryKey===null)
                        $this->$primaryKey=$builder->getLastInsertID($table);
                    elseif(is_array($primaryKey))
                    {
                        foreach($primaryKey as $pk)
                        {
                            if($this->$pk===null)
                            {
                                $this->$pk=$builder->getLastInsertID($table);
                                break;
                            }
                        }
                    }
                }
                $this->_pk=$this->getPrimaryKey();
                $this->afterSave();
                $this->setIsNewRecord(false);
                $this->setScenario('update');
                return true;
            }
        }
        return false;
    }
    
    public function update($attributes=null)
    {
        if($this->getIsNewRecord())
            throw new CDbException(Yii::t('yii','The active record cannot be updated because it is new.'));
        if($this->beforeSave())
        {
            Yii::trace(get_class($this).'.update()','system.db.ar.CActiveRecord');
            if($this->_pk===null)
                $this->_pk=$this->getPrimaryKey();
            // 间接调用updateByPk来完成操作
            $this->updateByPk($this->getOldPrimaryKey(),$this->getAttributes($attributes));
            $this->_pk=$this->getPrimaryKey();
            $this->afterSave();
            return true;
        }
        else
            return false;
    }

insert方法中调用的getCommandBuilder方法实现如下：

    :::php
    public function getCommandBuilder()
    {
        return $this->getDbConnection()->getSchema()->getCommandBuilder();
    }

其中getDbConnection方法实现如下：

    :::php
    public function getDbConnection()
    {
        if(self::$db!==null)
            return self::$db;
        else
        {
            self::$db=Yii::app()->getDb();
            if(self::$db instanceof CDbConnection)
                return self::$db;
            else
                throw new CDbException(Yii::t('yii','Active Record requires a "db" CDbConnection application component.'));
        }
    }

Yii中默认的DB组件名为db，所以如果你使用的是默认的db数据库，那么不用重写这个方法。否则，需要重写该方法，指明需要使用的数据库连接。

getDbConnection方法返回的是一个CDbConnection对象，其getSchema方法实现如下：

    :::php
    public function getSchema()
    {
        if($this->_schema!==null)
            return $this->_schema;
        else
        {
            // 返回数据库配置信息connectionString字段中的数据库驱动类型，如mysql
            $driver=$this->getDriverName();
            /*
                driverMap属性的定义：
                public $driverMap=array(
                  'pgsql'=>'CPgsqlSchema',    // PostgreSQL
                  'mysqli'=>'CMysqlSchema',   // MySQL
                  'mysql'=>'CMysqlSchema',    // MySQL
                  'sqlite'=>'CSqliteSchema',  // sqlite 3
                  'sqlite2'=>'CSqliteSchema', // sqlite 2
                  'mssql'=>'CMssqlSchema',    // Mssql driver on windows hosts
                  'dblib'=>'CMssqlSchema',    // dblib drivers on linux (and maybe others os) hosts
                  'sqlsrv'=>'CMssqlSchema',   // Mssql
                  'oci'=>'COciSchema',        // Oracle driver
               );
            */
            if(isset($this->driverMap[$driver]))
                // 加载对应的数据库驱动组件
                return $this->_schema=Yii::createComponent($this->driverMap[$driver], $this);
            else
                throw new CDbException(Yii::t('yii','CDbConnection does not support reading schema for {driver} database.',
                    array('{driver}'=>$driver)));
        }
    }

以mysql的CMysqlSchema为例，`Yii::createComponent($this->driverMap[$driver], $this)`一句会实例化yii/framework/db/schema/mysql/CMysqlSchema.php中定义的CMysqlSchema类。该类自身无构造方法，继承自抽象类CDbSchema，CDbSchema的构造方法实现如下：

    :::php
    public function __construct($conn)
    {
        // 保存着当前CDbConnection数据库连接对象
        $this->_connection=$conn;
        foreach($conn->schemaCachingExclude as $name)
            $this->_cacheExclude[$name]=true;
    }

得到CMysqlSchema对象后，调用其getCommandBuilder方法，该方法定义于类CDbSchema中，实现如下：

    :::php
    public function getCommandBuilder()
    {
        if($this->_builder!==null)
            return $this->_builder;
        else
            return $this->_builder=$this->createCommandBuilder();
    }

其中调用的方法createCommandBuilder实现如下：

    :::php
    protected function createCommandBuilder()
    {
        return new CDbCommandBuilder($this);
    }

实例化的CDbCommandBuilder类的构造方法如下所示：

    :::php
    public function __construct($schema)
    {
        $this->_schema=$schema;
        $this->_connection=$schema->getDbConnection();
    }

这样insert方法中调用的getCommandBuilder最终返回了一个CDbCommandBuilder对象。

而insert方法中`$table=$this->getMetaData()->tableSchema`一句调用的getMetaData方法实现如下：

    :::php
    public function getMetaData()
    {
        $className=get_class($this);
        if(!array_key_exists($className,self::$_md))
        {
            self::$_md[$className]=null; // preventing recursive invokes of {@link getMetaData()} via {@link __get()}
            self::$_md[$className]=new CActiveRecordMetaData($this);
        }
        return self::$_md[$className];
    }

其中实例化的类CActiveRecordMetaData，构造方法如下所示：

    :::php
    public function __construct($model)
    {
        // 当前model类名
        $this->_modelClassName=get_class($model);

        // 得到表名
        $tableName=$model->tableName();
        
        // 调用_schema（以mysql为例，其值为CMysqlSchema对象）的getTable方法
        if(($table=$model->getDbConnection()->getSchema()->getTable($tableName))===null)
            throw new CDbException(Yii::t('yii','The table "{table}" for active record class "{class}" cannot be found in the database.',
                array('{class}'=>$this->_modelClassName,'{table}'=>$tableName)));
        if($table->primaryKey===null)
        {
            $table->primaryKey=$model->primaryKey();
            if(is_string($table->primaryKey) && isset($table->columns[$table->primaryKey]))
                $table->columns[$table->primaryKey]->isPrimaryKey=true;
            elseif(is_array($table->primaryKey))
            {
                foreach($table->primaryKey as $name)
                {
                    if(isset($table->columns[$name]))
                        $table->columns[$name]->isPrimaryKey=true;
                }
            }
        }
        // 将数据表结构信息存于属性tableSchema
        $this->tableSchema=$table;
        $this->columns=$table->columns;

        foreach($table->columns as $name=>$column)
        {
            if(!$column->isPrimaryKey && $column->defaultValue!==null)
                $this->attributeDefaults[$name]=$column->defaultValue;
        }

        foreach($model->relations() as $name=>$config)
        {
            $this->addRelation($name,$config);
        }
    }
    
其中调用的getTable方法，定义于抽象类CDbSchema中，实现如下：

    :::php
    public function getTable($name,$refresh=false)
    {
        if($refresh===false && isset($this->_tables[$name]))
            return $this->_tables[$name];
        else
        {
            if($this->_connection->tablePrefix!==null && strpos($name,'{{')!==false)
                $realName=preg_replace('/\{\{(.*?)\}\}/',$this->_connection->tablePrefix.'$1',$name);
            else
                $realName=$name;

            // temporarily disable query caching
            if($this->_connection->queryCachingDuration>0)
            {
                $qcDuration=$this->_connection->queryCachingDuration;
                $this->_connection->queryCachingDuration=0;
            }
            
            // 先尝试从缓存中取数据表结构信息
            // CDbConnection类的schemaCachingDuration属性默认为0，如果不配置该属性，那么就不会使用缓存，那么每次增、删、改、查操作都需要loadTable，
            // 对数据库的压力，以及性能影响是不是很大？！但如果加了缓存，那么当对数据表的结构做变更时会不会有问题？
            if(!isset($this->_cacheExclude[$name]) && ($duration=$this->_connection->schemaCachingDuration)>0 && $this->_connection->schemaCacheID!==false && ($cache=Yii::app()->getComponent($this->_connection->schemaCacheID))!==null)
            {
                $key='yii:dbschema'.$this->_connection->connectionString.':'.$this->_connection->username.':'.$name;
                $table=$cache->get($key);
                // 没取到或者需要刷新缓存，则从数据库获取，并更新缓存
                if($refresh===true || $table===false)
                {
                    $table=$this->loadTable($realName);
                    if($table!==null)
                        $cache->set($key,$table,$duration);
                }
                $this->_tables[$name]=$table;
            }
            else
                // 直接从数据库获取数据表结构信息
                $this->_tables[$name]=$table=$this->loadTable($realName);

            if(isset($qcDuration))  // re-enable query caching
                $this->_connection->queryCachingDuration=$qcDuration;

            return $table;
        }
    }

其中调用的loadTable方法最终是通过执行SQL语句：

- `'SHOW FULL COLUMNS FROM '.$table->rawName`
- `'SHOW CREATE TABLE '.$table->rawName`

来获取数据表信息，并将信息存于一个CMysqlColumnSchema对象中（以mysql为例）。

insert方法中`$command=$builder->createInsertCommand($table,$this->getAttributes($attributes))`一句createInsertCommand方法定义于CDbCommandBuilder类中，实现如下：

    :::php
    public function createInsertCommand($table,$data)
    {
        $this->ensureTable($table);
        $fields=array();
        $values=array();
        $placeholders=array();
        $i=0;
        foreach($data as $name=>$value)
        {
            if(($column=$table->getColumn($name))!==null && ($value!==null || $column->allowNull))
            {
                $fields[]=$column->rawName;
                if($value instanceof CDbExpression)
                {
                    $placeholders[]=$value->expression;
                    foreach($value->params as $n=>$v)
                        $values[$n]=$v;
                }
                else
                {
                    $placeholders[]=self::PARAM_PREFIX.$i;
                    $values[self::PARAM_PREFIX.$i]=$column->typecast($value);
                    $i++;
                }
            }
        }
        if($fields===array())
        {
            $pks=is_array($table->primaryKey) ? $table->primaryKey : array($table->primaryKey);
            foreach($pks as $pk)
            {
                $fields[]=$table->getColumn($pk)->rawName;
                $placeholders[]=$this->getIntegerPrimaryKeyDefaultValue();
            }
        }
        $sql="INSERT INTO {$table->rawName} (".implode(', ',$fields).') VALUES ('.implode(', ',$placeholders).')';
        $command=$this->_connection->createCommand($sql);

        foreach($values as $name=>$value)
            $command->bindValue($name,$value);

        return $command;
    }

另外还有与update、delete等操作对应的方法createUpdateCommand、createDeleteCommand等。

update方法与insert方法的逻辑基本是一致的。

CActiveRecord中所有数据库增、删、改、查操作，在构建出目标sql语句后，都是调用CDbConnection类的方法createCommand来得到一个CDbCommand类的对象，最后调用该对象的execute、query、queryAll等方法来完成查询获取结果。

------

对于model类实例对象的属性赋值，依赖于CActiveRecord类的方法__set，实现如下：

    :::php
    public function __set($name,$value)
    {
        if($this->setAttribute($name,$value)===false)
        {
            if(isset($this->getMetaData()->relations[$name]))
                $this->_related[$name]=$value;
            else
                parent::__set($name,$value);
        }
    }

其中调用的setAttribute方法的实现如下：

    :::php
    public function setAttribute($name,$value)
    {
        if(property_exists($this,$name))
            $this->$name=$value;
        elseif(isset($this->getMetaData()->columns[$name]))
            $this->_attributes[$name]=$value;
        else
            return false;
        return true;
    }

------

基于CActiveRecord类，如果想进行读取操作，那么子类需要重写model方法。CActiveRecord类的model方法实现如下：

    /**
     * Returns the static model of the specified AR class.
     * The model returned is a static instance of the AR class.
     * It is provided for invoking class-level methods (something similar to static class methods.)
     *
     * EVERY derived AR class must override this method as follows,
     * <pre>
     * public static function model($className=__CLASS__)
     * {
     *     return parent::model($className);
     * }
     * </pre>
     *
     * @param string $className active record class name.
     * @return CActiveRecord active record model instance.
     */
    public static function model($className=__CLASS__)
    {
        if(isset(self::$_models[$className]))
            return self::$_models[$className];
        else
        {
            $model=self::$_models[$className]=new $className(null);
            $model->attachBehaviors($model->behaviors());
            return $model;
        }
    }

根据该方法的注释可知道，所有的子类必须如下重写model方法：

    :::php
    public static function model($className = __CLASS__)
    {
        return parent::model($className);
    }

为什么必须重写model方法呢？因为`__CLASS__`指的并不是当前对象所属的类，而是方法定义所在的类。

隔壁的哥们告诉我，在 PHP 5.3 之后，如果CActiveRecord的model方法如下实现，就可以不用这样使用需要重写了。

    :::php
    public static function model()
    {
        $className = get_called_class();
        if(isset(self::$_models[$className]))
            return self::$_models[$className];
        else
        {
            $model=self::$_models[$className]=new $className(null);
            $model->attachBehaviors($model->behaviors());
            return $model;
        }
    }

在得到$model后，就可以调用对象的query、find、findAll、findByPk、findAllByPk、findByAttributes、findAllByAttributes、findBySql、findAllBySql等方法来查询数据。其中find、findAll、findByPk、findAllByPk、findByAttributes、findAllByAttributes最终是通过调用query方法来实现查询的。query方法的第一个参数是一个CDbCriteria类实例对象，这就意味着调用query方法来实现查询的方法需要根据参数实例化一个CDbCriteria类对象。如find方法实现如下：

    :::php
    /**
     * Finds a single active record with the specified condition.
     * @param mixed $condition query condition or criteria.
     * If a string, it is treated as query condition (the WHERE clause);
     * If an array, it is treated as the initial values for constructing a {@link CDbCriteria} object;
     * Otherwise, it should be an instance of {@link CDbCriteria}.
     * @param array $params parameters to be bound to an SQL statement.
     * This is only used when the first parameter is a string (query condition).
     * In other cases, please use {@link CDbCriteria::params} to set parameters.
     * @return CActiveRecord the record found. Null if no record is found.
     */
    public function find($condition='',$params=array())
    {
        Yii::trace(get_class($this).'.find()','system.db.ar.CActiveRecord');
        // 实例化一个CDbCriteria对象
        $criteria=$this->getCommandBuilder()->createCriteria($condition,$params);
        return $this->query($criteria);
    }

而query方法的实现如下所示：

    :::php
    protected function query($criteria,$all=false)
    {
        $this->beforeFind();
        $this->applyScopes($criteria);

        if(empty($criteria->with))
        {
            if(!$all)
                $criteria->limit=1;
            // createFindCommand在上面提过
            $command=$this->getCommandBuilder()->createFindCommand($this->getTableSchema(),$criteria,$this->getTableAlias());
            return $all ? $this->populateRecords($command->queryAll(), true, $criteria->index) : $this->populateRecord($command->queryRow());
        }
        else
        {
            $finder=$this->getActiveFinder($criteria->with);
            return $finder->query($criteria,$all);
        }
    }

------

对于查询操作，很多时候需要多表关联查询。那么基于CActiveRecord如何实现多表关联查询（隐式自动地）呢？

CActiveRecord类有个方法relations()：

    :::php
    public function relations()
    {
        return array();
    }

这个方法的注释非常长，说明如何通过该方法声明当前model对应的数据表有哪些关联数据表，是何种关联关系。继承自CActiveRecord类的model类若想使用隐式的多表关联查询，则需要重写该方法。

举例来说，有数据表UserContacts、Users，UserContacts中有外键字段user_id关联到Users。如果基于CActiveRecord在查询UserContacts记录时，需要便捷地获取关联Users的记录。那么可以在UserContacts数据表对应的model类中，这样重写relations方法：

    :::php
    public function relations()
    {
        return array(
                'user' => array(self::BELONGS_TO, 'Users', 'user_id'),
        );
    }

那么在得到一条记录对象（$uc）时，通过调用$uc->user，即可得到与该UserContacts记录关联的Users记录。不过这条关联记录的获取可能是在调用$uc->user时才去查询数据库的。

若想提前将关联记录查询出来准备好，则可以再调用find、findAll等查询方法之前先调用with方法，如`self::model()->with('user')->find(array('usercontact_id' => 1))`，或者这样调用find方法`self::model()->find(array('usercontact_id' => 1, 'with' => 'user'))`。

那么在调用`$uc->user`时，如何知道user是一个关联项，而不是一个当前对象的属性？如果当前对象对应的数据表已有一个名为user的字段，是否会屏蔽掉关联项？且看CActiveRecord类的__get方法：

    :::php
    public function __get($name)
    {
        // 先查看当前model类对应的数据表是否有名为$name的字段
        if(isset($this->_attributes[$name]))
            return $this->_attributes[$name];
        elseif(isset($this->getMetaData()->columns[$name]))
            return null;
        // 查看是否有名为$name的关联项
        elseif(isset($this->_related[$name]))
            return $this->_related[$name];
        elseif(isset($this->getMetaData()->relations[$name]))
            return $this->getRelated($name);
        else
            return parent::__get($name);
    }

------

*注：本文为草稿状态*