1 设置`SstWriteOptions`，涉及`StorageFormatHint`每个`row_group`涉及的`num_rows_per_row_group`，压缩格式，`max_buffer_size`
2 设置`ObjectStoreRef`，测试使用本地存储
3 创建要写入表的`schema`
  - 创建`schema_builder`,在其中使用`column_schema_builder`调用`build`用于生成对应的列，有`key column`和`normal column`的区别
  - 其中`key column`会记录对应时间列的`index`同时会维护`key column`的`index`
  - 随后`schema_builder`调用`build`，`build`中会进行时间列的确认(这里存在`tsid`的时候为什么`key column`一定要等于2)
  - 确认完后在`build`中会对每个`column schema`调用`to_arrow_field`将`column`转换为`arrow::datatypes::Field`,结构如下
    ```   
    pub struct Field {
        name: String, 
        data_type: DataType,
        nullable: bool,
        dict_id: i64,
        dict_is_ordered: bool,
        /// A map of key-value pairs containing additional custom meta data.
        metadata: HashMap<String, String>,
    }
    ```
    因此转换过程首先是设置`metadata`，ceresdb中涉及3个用户定义的metadata，为`enum ArrowFieldMetaKey {Id,IsTag,Comment}`,分别会初始化为schema的id,schema中存储的是否是tag列以及schema中的comment（comment有啥用？）。
    随后设置Field的name，data_type需要将ceresdb的type转换成datafusion的。（dict_id和dict_id_ordered是干啥的？好像只在IPC中使用，用于匹配字典？）
    - 随后生成ArrowSchemaRef，其中也涉及用户定义的metadata，为`enum ArrowSchemaMetaKey {PrimaryKeyIndexes,TimestampIndex,Version,}`,其中PrimaryKeyIndexes是所有key column的index集合，TimestampIndex是刚刚记录的timestamp的index，version是schema的版本号
    - 最后使用上述生成的相关变量返回schema
  - 对于刚刚生成要写入的表的schema，单测中使用的是(key1(varbinary), key2(timestamp), field1(double), field2(string), field3(date), field4(time))
4 生成`MetaData`,其实就是`zonemap`，维护最大最小值，时间范围等  
5 写入接口是流式的，我们需要生成数据流，流的每次Batch生成过程如下:
  - 测试会调用build_row生成对应的一行数据，一行数据是一个Row结构，Row里维护了` cols: Vec<Datum>,`,`Datum`是枚举，存了每种数据类型对应的数据。
  - 测试会调用`build_record_batch_with_key`用于生成`RecordBatchWithKey`。
    - 首先会生成`projected_schema`。需要生成`RecordSchemaWithKey`:传入schema和需要projection的index，`key column`仍然需要全部加入，但是非`key column`只会加入需要projection的index的col,使用以上筛选过的column生成新的`RecordSchema`(其实就是将之前的schema看做全集，挑选出必须包含key column的子集),最后返回`RecordSchemaWithKey`。
    - 生成`RecordSchema`同理，只不过不需要包含所有`key column`，只需要包含需要`projection`的column,最后使用以上变量生成`projected_schema`
    - 随后使用刚刚生成的`projected_schema`和原始`schema`生成`RowProjector`,对于每个`column`需要验证版本号，判断兼容性，生成最后的需要`projection`的`index`,最后生成`RowProjector`。
    - 调用`RecordBatchWithKeyBuilder::with_capacity`使用`projected_schema`的`RecordSchemaWithKey`和指定`capacity`（capacity是需要预分配的item数量？）生成`RecordSchemaWithKey`中所有`column`的`ColumnBlockBuilder`,最后使用这些builder和recordschemawithkey生成RecordBatchWithKeyBuilder
    - 生成`IndexInWriterSchema`（这个有啥用）
    - 对于`rows`中的每一个`row`，使用如下过程
        - 传入`buf`、`schema`和`index_in_writer`生成`ContiguousRowWriter`,
        - 调用`writer`的`write_row`方法，把当前`row`的值序列化到buf中
        - 使用schema生成ContiguousRowReader，使用RowProjector和ContiguousRowReader生成ProjectedContiguousRow
        - 调用builder的`append_projected_contiguous_row`，对于每个`builder`中的每个`ColumnBlockBuilder`，取出在`row`中的数据，使用`append_view`加入到`ColumnBlockBuilder`中。`append_view`是个宏，对于每个类型有不同的处理，例如如果是string类型则会使用arrow的stringbuilder 的 builder.append_value方法，
    - 最后会调用build,其中每个ColumnBlockBuilder会调用builder.finsh用于生成arrow格式的数组，随后和arrow_schema一起生成recordBatch

6 使用上述配置生成`ParquetSstWriter`，随后调用`write`方法实际写入parquet，流程如下:
  - 使用传入的配置生成`RecordBatchGroupWriter`
  - 调用`ObjectStoreMultiUploadAborter::initialize_upload`, 这是干啥的？
  - 使用刚刚生成的`RecordBatchGroupWriter`的write_all方法,流程如下:
    - 生成`ParquetEncoder`，根据是否需要`filter`生成`parquet_filter`
    - 进入循环，在流中每次取最多`num_rows_per_row_group`条数据，随后使用取出数据构造filter，把row_group中的相关数据封装到`arrow_row_group`中，调用`ParquetEncoder`对应的encode方法，把`arrow_row_group`数组合并成一个`RecordBatch`后调用`AsyncArrowWriter`进行写入
    - 将上述`metadata`(也就是`zonemap`)设置到`ParquetEncoder`中，会调用`AsyncArrowWriter`的`append_key_value_metadata`方法
    - `ParquetEncoder`调用`close`，生成最终的`parquet`文件



