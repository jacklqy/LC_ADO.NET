# LC_ADO.NET
数据库操作之ADO.NET五大对象详解：Connection 连接对象、Command 命令对象 、SqlDataReader 数据读取对象、DataSet 数据集对象 、SqlDataAdapter 数据适配器对象 

## Connection 连接对象 

用于对数据库的连接操作。传入的参数为连接字符串。

## Command 命令对象 

用于执行对数据库的操作 ，传入的参数可以为字符串或存储过程，也必须传入连接对象的实例Connection。

## SqlDataReader 数据读取对象

用于对数据的读取操作，实例.Read()方法可以返回一个True或者False来判断是否读取到了数据,我们可以直接使用 实例[“字段名称”]来取出读取到的数据。

## DataSet 数据集对象 

该对象类似于在内存中的多张虚拟的表，我们可以动态的添加行，列，数据，对数据库进行更新回传操作。

## SqlDataAdapter 数据适配器对象 

该对象可用于数据库的增删改差操作，一次性将读取到的内容加载到内存中，可以脱离连接进行操作，返回到一个DataSet对象

 

## SqlDataReader和SqlDataAdapter读取数据的不同

DataReader 实现对数据的读取时需要连接着数据库，每次只能读取到一条数据，是一种只进流的读取，也就是当我读取到了一条数据，就只能接着读取下一条数据，不能再次读取这条数据了。

DataApater 实现对数据的读取时，是一次性将读取到的整张或多张表加载到内存中，比较消耗内存，不需要再连接着数据库。我们可以借助DataSet对象来将读取到的表加载到DataSet中，就像对表的操作一样，我们可以获取它的行和列来进行操作。

## SqlHelper
```
using System;
using System.Collections.Generic;
using System.Configuration;
using System.Data;
using System.Data.SqlClient;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace DbUtility
{
    public class SqlHelper
    {
        /// <summary>
        /// 连接字符串
        /// </summary>
        private static readonly string connStr = ConfigurationManager.ConnectionStrings["connStr"].ConnectionString;
        /// <summary>
        /// 增、删、改的通用方法
        /// 执行Sql语句或存储过程，返回受影响的行数
        /// SQL注入 
        /// </summary>
        /// <param name="sql">sql语句或存储过程名</param>
        /// <param name="cmdType">执行的脚本类型 1:sql语句  2:存储过程</param>
        /// <param name="parameters">参数列表</param>
        /// <returns></returns>
        public static int ExecuteNonQuery(string sql, int cmdType, params SqlParameter[] parameters)
        {
            //select @@Identity 返回上一次插入记录时自动产生的ID
            int result = 0;//返回结果
            using (SqlConnection conn = new SqlConnection(connStr))
            {
                //执行脚本的对象cmd
                SqlCommand cmd = BuilderCommand(conn, sql, cmdType, null, parameters);
                result = cmd.ExecuteNonQuery();//执行T-SQL并返回受影响行数
                cmd.Parameters.Clear();
            }
            //using原理：类似于try finally
            return result;
        }

        /// <summary>
        /// 执行sql查询，返回第一行第一列的值
        /// </summary>
        /// <param name="sql">sql语句或存储过程</param>
        /// <param name="cmdType">执行的脚本类型 1:sql语句  2:存储过程</param>
        /// <param name="parameters">参数列表</param>
        /// <returns></returns>
        public static object ExecuteScalar(string sql, int cmdType, params SqlParameter[] parameters)
        {
            object result = null;//返回结果
            using (SqlConnection conn = new SqlConnection(connStr))
            {
                //执行脚本的对象cmd
                SqlCommand cmd = BuilderCommand(conn, sql, cmdType, null, parameters);
                result = cmd.ExecuteScalar();//执行T-SQL并返回第一行第一列的值
                cmd.Parameters.Clear();
                if (result == null || result == DBNull.Value)
                {
                    return null;
                }
                else
                {
                    return result;
                }
            }
        }

        /// <summary>
        /// 执行sql查询,返回SqlDataReader对象
        /// </summary>
        /// <param name="sql">sql语句或存储过程</param>
        /// <param name="cmdType">执行的脚本类型 1:sql语句  2:存储过程</param>
        /// <param name="parameters">参数列表</param>
        /// <returns></returns>
        public static SqlDataReader ExecuteReader(string sql, int cmdType, params SqlParameter[] parameters)
        {
            SqlConnection conn = new SqlConnection(connStr);
            SqlCommand cmd = BuilderCommand(conn, sql, cmdType, null, parameters);
            SqlDataReader reader;
            try
            {
                //conn.Open();
                reader = cmd.ExecuteReader(CommandBehavior.CloseConnection);
                return reader;
            }
            catch (Exception ex)
            {
                conn.Close();
                throw new Exception("创建reader对象发生异常", ex);
            }

        }

        /// <summary>
        /// 执行查询，查询结果填充到DataTable 只针对查询一个表的情况
        /// </summary>
        /// <param name="sql">sql语句或存储过程</param>
        /// <param name="cmdType">执行的脚本类型 1:sql语句  2:存储过程</param>
        /// <param name="parameters">参数列表</param>
        /// <returns></returns>
        public static DataTable GetDataTable(string sql, int cmdType, params SqlParameter[] parameters)
        {
            DataTable dt = null;
            using (SqlConnection conn = new SqlConnection(connStr))
            {
                SqlCommand cmd = BuilderCommand(conn, sql, cmdType, null, parameters);
                SqlDataAdapter da = new SqlDataAdapter(cmd);
                dt = new DataTable();
                da.Fill(dt);
                cmd.Parameters.Clear();
            }
            return dt;
        }

        /// <summary>
        /// 执行查询，数据填充到DataSet
        /// </summary>
        /// <param name="sql">sql语句或存储过程</param>
        /// <param name="cmdType">执行的脚本类型 1:sql语句  2:存储过程</param>
        /// <param name="parameters">参数列表</param>
        /// <returns></returns>
        public static DataSet GetDataSet(string sql, int cmdType, params SqlParameter[] parameters)
        {
            DataSet ds = null;
            using (SqlConnection conn = new SqlConnection(connStr))
            {
                SqlCommand cmd = BuilderCommand(conn, sql, cmdType, null, parameters);
                //数据适配器
                //conn 自动打开  断开式连接
                SqlDataAdapter da = new SqlDataAdapter(cmd);
                ds = new DataSet();
                da.Fill(ds);
                cmd.Parameters.Clear();
                //自动关闭conn
            }
            return ds;
        }

        /// <summary>
        /// 事务 执行批量sql
        /// </summary>
        /// <param name="listSql"></param>
        /// <returns></returns>
        public static bool ExecuteTrans(List<string> listSql)
        {
            using (SqlConnection conn = new SqlConnection(connStr))
            {
                conn.Open();
                SqlTransaction trans = conn.BeginTransaction();
                SqlCommand cmd = BuilderCommand(conn, "", 1, trans);
                try
                {
                    int count = 0;
                    for (int i = 0; i < listSql.Count; i++)
                    {
                        if (listSql[i].Length > 0)
                        {
                            cmd.CommandText = listSql[i];
                            cmd.CommandType = CommandType.Text;
                            count += cmd.ExecuteNonQuery();
                        }
                    }
                    trans.Commit();
                    return true;
                }
                catch (Exception ex)
                {
                    trans.Rollback();
                    throw new Exception("执行事务出现异常", ex);
                }
            }
        }

        /// <summary>
        /// 事务 批量执行 CommandInfo 包括sql,脚本类型，参数列表
        /// </summary>
        /// <param name="comList"></param>
        /// <returns></returns>
        public static bool ExecuteTrans(List<CommandInfo> comList)
        {
            using (SqlConnection conn = new SqlConnection(connStr))
            {
                conn.Open();
                SqlTransaction trans = conn.BeginTransaction();
                SqlCommand cmd = BuilderCommand(conn, "", 1, trans);
                try
                {
                    int count = 0;
                    for (int i = 0; i < comList.Count; i++)
                    {
                        cmd.CommandText = comList[i].CommandText;

                        if (comList[i].IsProc)
                            cmd.CommandType = CommandType.StoredProcedure;
                        else
                            cmd.CommandType = CommandType.Text;

                        if (comList[i].Paras != null && comList[i].Paras.Length > 0)
                        {
                            cmd.Parameters.Clear();
                            foreach (var p in comList[i].Paras)
                            {
                                cmd.Parameters.Add(p);
                            }
                        }
                        count += cmd.ExecuteNonQuery();

                    }
                    trans.Commit();
                    return true;
                }
                catch (Exception ex)
                {
                    trans.Rollback();
                    throw new Exception("执行事务出现异常", ex);
                }
            }
        }

        //public static T ExecuteSql<T>(string sql, int cmdType, DbParameter[] paras, Func<IDbCommand, T> action)
        //{
        //    using (DbConnection conn = new SqlConnection(connStr))
        //    {
        //        conn.Open();
        //        IDbCommand cmd = conn.CreateCommand();
        //        cmd.CommandText = sql;
        //        if (cmdType==2)
        //            cmd.CommandType = CommandType.StoredProcedure;
        //        return action(cmd);
        //    }
        //}

        public static T ExecuteTrans<T>(Func<IDbCommand, T> action)
        {
            using (IDbConnection conn = new SqlConnection(connStr))
            {
                conn.Open();
                IDbTransaction trans = conn.BeginTransaction();
                IDbCommand cmd = conn.CreateCommand();
                cmd.Transaction = trans;
                return action(cmd);
            }
        }




        /// <summary>
        /// 构建SqlCommand
        /// </summary>
        /// <param name="conn">数据库连接对象</param>
        /// <param name="sql">SQL语句或存储过程</param>
        /// <param name="comType">命令字符串的类型</param>
        /// <param name="trans">事务</param>
        /// <param name="paras">参数数组</param>
        /// <returns></returns>
        private static SqlCommand BuilderCommand(SqlConnection conn, string sql, int cmdType, SqlTransaction trans, params SqlParameter[] paras)
        {
            if (conn == null) throw new ArgumentNullException("连接对象不能为空！");
            SqlCommand command = new SqlCommand(sql, conn);
            if (cmdType == 2)
                command.CommandType = CommandType.StoredProcedure;
            if (conn.State == ConnectionState.Closed)
                conn.Open();
            if (trans != null)
                command.Transaction = trans;
            if (paras != null && paras.Length > 0)
            {
                command.Parameters.Clear();
                command.Parameters.AddRange(paras);
            }
            return command;
        }

        /// <summary>
        /// 添加参数集合
        /// </summary>
        /// <param name="cmd"></param>
        /// <param name="paras"></param>
        public static void AddParas(IDbCommand cmd, SqlParameter[] paras)
        {
            cmd.Parameters.Clear();
            if(paras!=null &&paras.Length >0)
            {
                foreach(SqlParameter p in paras)
                {
                    cmd.Parameters.Add(p);
                }
            }
        }

    }
}

```
