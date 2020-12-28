***
Spring-jdbc异常处理
***
>>1、为什么多种数据库异常（数据库不一样Mysql、H2等），最后对应的Spring-jdbc的异常是一样的？

>>2、怎样自定义一个Sql异常？


>>------------------------------------

1、Spring-jdbc Sql异常继承关系
 
      a、org.springframework.dao.DataAccessException
    
      Spring自定义的sql异常基类，Spring里面所有Sql异常都是继承此基类
    
     b、继承关系

2、异常转换
    
   a、SQLExceptionTranslator 
        
        这是Spring的Sql异常转换接口
        核心方法：protected DataAccessException doTranslate(String task, @Nullable String sql, SQLException ex)
   ```java
   protected DataAccessException doTranslate(String task, @Nullable String sql, SQLException ex) {
		SQLException sqlEx = ex;
		if (sqlEx instanceof BatchUpdateException && sqlEx.getNextException() != null) {
			SQLException nestedSqlEx = sqlEx.getNextException();
			if (nestedSqlEx.getErrorCode() > 0 || nestedSqlEx.getSQLState() != null) {
				sqlEx = nestedSqlEx;
			}
		}

		// First, try custom translation from overridden method.
		DataAccessException dae = customTranslate(task, sql, sqlEx);
		if (dae != null) {
			return dae;
		}

		// Next, try the custom SQLException translator, if available.
		SQLErrorCodes sqlErrorCodes = getSqlErrorCodes();
		if (sqlErrorCodes != null) {
			SQLExceptionTranslator customTranslator = sqlErrorCodes.getCustomSqlExceptionTranslator();
			if (customTranslator != null) {
				DataAccessException customDex = customTranslator.translate(task, sql, sqlEx);
				if (customDex != null) {
					return customDex;
				}
			}
		}

		// Check SQLErrorCodes with corresponding error code, if available.
		if (sqlErrorCodes != null) {
			String errorCode;
			if (sqlErrorCodes.isUseSqlStateForTranslation()) {
				errorCode = sqlEx.getSQLState();
			}
			else {
				// Try to find SQLException with actual error code, looping through the causes.
				// E.g. applicable to java.sql.DataTruncation as of JDK 1.6.
				SQLException current = sqlEx;
				while (current.getErrorCode() == 0 && current.getCause() instanceof SQLException) {
					current = (SQLException) current.getCause();
				}
				errorCode = Integer.toString(current.getErrorCode());
			}

			if (errorCode != null) {
				// Look for defined custom translations first.
				CustomSQLErrorCodesTranslation[] customTranslations = sqlErrorCodes.getCustomTranslations();
				if (customTranslations != null) {
					for (CustomSQLErrorCodesTranslation customTranslation : customTranslations) {
						if (Arrays.binarySearch(customTranslation.getErrorCodes(), errorCode) >= 0 &&
								customTranslation.getExceptionClass() != null) {
							DataAccessException customException = createCustomException(
									task, sql, sqlEx, customTranslation.getExceptionClass());
							if (customException != null) {
								logTranslation(task, sql, sqlEx, true);
								return customException;
							}
						}
					}
				}
				// Next, look for grouped error codes.
				if (Arrays.binarySearch(sqlErrorCodes.getBadSqlGrammarCodes(), errorCode) >= 0) {
					logTranslation(task, sql, sqlEx, false);
					return new BadSqlGrammarException(task, (sql != null ? sql : ""), sqlEx);
				}
				else if (Arrays.binarySearch(sqlErrorCodes.getInvalidResultSetAccessCodes(), errorCode) >= 0) {
					logTranslation(task, sql, sqlEx, false);
					return new InvalidResultSetAccessException(task, (sql != null ? sql : ""), sqlEx);
				}
				else if (Arrays.binarySearch(sqlErrorCodes.getDuplicateKeyCodes(), errorCode) >= 0) {
					logTranslation(task, sql, sqlEx, false);
					return new DuplicateKeyException(buildMessage(task, sql, sqlEx), sqlEx);
				}
				else if (Arrays.binarySearch(sqlErrorCodes.getDataIntegrityViolationCodes(), errorCode) >= 0) {
					logTranslation(task, sql, sqlEx, false);
					return new DataIntegrityViolationException(buildMessage(task, sql, sqlEx), sqlEx);
				}
				else if (Arrays.binarySearch(sqlErrorCodes.getPermissionDeniedCodes(), errorCode) >= 0) {
					logTranslation(task, sql, sqlEx, false);
					return new PermissionDeniedDataAccessException(buildMessage(task, sql, sqlEx), sqlEx);
				}
				else if (Arrays.binarySearch(sqlErrorCodes.getDataAccessResourceFailureCodes(), errorCode) >= 0) {
					logTranslation(task, sql, sqlEx, false);
					return new DataAccessResourceFailureException(buildMessage(task, sql, sqlEx), sqlEx);
				}
				else if (Arrays.binarySearch(sqlErrorCodes.getTransientDataAccessResourceCodes(), errorCode) >= 0) {
					logTranslation(task, sql, sqlEx, false);
					return new TransientDataAccessResourceException(buildMessage(task, sql, sqlEx), sqlEx);
				}
				else if (Arrays.binarySearch(sqlErrorCodes.getCannotAcquireLockCodes(), errorCode) >= 0) {
					logTranslation(task, sql, sqlEx, false);
					return new CannotAcquireLockException(buildMessage(task, sql, sqlEx), sqlEx);
				}
				else if (Arrays.binarySearch(sqlErrorCodes.getDeadlockLoserCodes(), errorCode) >= 0) {
					logTranslation(task, sql, sqlEx, false);
					return new DeadlockLoserDataAccessException(buildMessage(task, sql, sqlEx), sqlEx);
				}
				else if (Arrays.binarySearch(sqlErrorCodes.getCannotSerializeTransactionCodes(), errorCode) >= 0) {
					logTranslation(task, sql, sqlEx, false);
					return new CannotSerializeTransactionException(buildMessage(task, sql, sqlEx), sqlEx);
				}
			}
		}

		// We couldn't identify it more precisely - let's hand it over to the SQLState fallback translator.
		if (logger.isDebugEnabled()) {
			String codes;
			if (sqlErrorCodes != null && sqlErrorCodes.isUseSqlStateForTranslation()) {
				codes = "SQL state '" + sqlEx.getSQLState() + "', error code '" + sqlEx.getErrorCode();
			}
			else {
				codes = "Error code '" + sqlEx.getErrorCode() + "'";
			}
			logger.debug("Unable to translate SQLException with " + codes + ", will now try the fallback translator");
		}

		return null;
	}
   ```
   b、两个重要的实现类
       
       （1）、SQLErrorCodeSQLExceptionTranslator
          该实现类是将定义好的数据库异常码转换到对应的Spring的异常，因此可以做到准确的异常匹配
       （2）、SQLStateSQLExceptionTranslator
          该实现类是将java.sql.SQLException转换为Spring的异常，因此准确性不是很高，依赖于java.sql.SQLException

3、Spring-jdbc 对各个数据库异常码的维护
 
    a、SQLErrorCodes 
    b、SQLErrorCodesFactory & sql-error-codes.xml