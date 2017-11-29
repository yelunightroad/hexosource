---
title: yii1.1版本下访问数据库时需要注意的问题
date: 2017-11-26 19:42:55
tags: [yii,greenplum,emulate prepare]
categories:
- web开发
- 数据库访问
---

上篇文章提到我们使用了greenplum来代替mysql来适应数据规模较大的情况，同时尽量保证小的改动。服务端我们使用的是yii1.1版本，之所以不是2.0原因很简单，历史选择。但在使用greenplum的过程中我们发现某些查询下速度特别慢，后经过各种测试发现，一旦调用了prepare方法，在数据量较大时速度就会特别慢，怀疑时greenplum优化的问题，但由于使用的是dba自己改造后的版本，也就没有深究其官方版本是不是也存在这样的bug。

那么现在的问题就变成了我们使用的是CDbcommand模块queryAll方法，并没有调用其prepare方法，那么为什么还会慢的，阅读其源码发现，所有的query方法，最终都是靠queryInternal方法实现的，在该函数的32行我们不难发现，默认情况先总会调用prepare函数，所以即使我们没有显示的调用CDbcommand的prepare方法，最终还是难以避免这一点。

```php
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
		if($this->_connection->queryCachingCount>0 && $method!==''
				&& $this->_connection->queryCachingDuration>0
				&& $this->_connection->queryCacheID!==false
				&& ($cache=Yii::app()->getComponent($this->_connection->queryCacheID))!==null)
		{
			$this->_connection->queryCachingCount--;
			$cacheKey='yii:dbquery'.':'.$method.':'.$this->_connection->connectionString.':'.$this->_connection->username;
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
			if($method==='')
				$result=new CDbDataReader($this);
			else
			{
				$mode=(array)$mode;
				call_user_func_array(array($this->_statement, 'setFetchMode'), $mode);
				$result=$this->_statement->$method();
				$this->_statement->closeCursor();
			}
			if($this->_connection->enableProfiling)
				Yii::endProfile('system.db.CDbCommand.query('.$this->getText().$par.')','system.db.CDbCommand.query');
			if(isset($cache,$cacheKey))
				$cache->set($cacheKey, array($result), $this->_connection->queryCachingDuration, $this->_connection->queryCachingDependency);
			return $result;
		}
		catch(Exception $e)
		{
			if($this->_connection->enableProfiling)
				Yii::endProfile('system.db.CDbCommand.query('.$this->getText().$par.')','system.db.CDbCommand.query');
			$errorInfo=$e instanceof PDOException ? $e->errorInfo : null;
			$message=$e->getMessage();
			Yii::log(Yii::t('yii','CDbCommand::{method}() failed: {error}. The SQL statement executed was: {sql}.',
				array('{method}'=>$method, '{error}'=>$message, '{sql}'=>$this->getText().$par)),CLogger::LEVEL_ERROR,'system.db.CDbCommand');
			if(YII_DEBUG)
				$message.='. The SQL statement executed was: '.$this->getText().$par;
			throw new CDbException(Yii::t('yii','CDbCommand failed to execute the SQL statement: {error}',
				array('{error}'=>$message)),(int)$e->getCode(),$errorInfo);
		}
	}

```

最终的解决方案是在配置中加了'emulatePrepare' => true的配置，该配置项其实就是启用了pdo中的模拟预处理，用模拟预处理代替greenplum龟速的预处理，最终解决了该问题。