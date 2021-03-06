# Data List Converter

Data List Converter is a tool for converting data between different formats.

Example:

```ruby
data = [{name: 'James', age: '22'}, {name: 'Bob', age: '33'}]
DataListConverter.convert(:item_data, :table_data, data)
# => [["name", "age"], ["James", "22"], ["Bob", "33"]] 

require 'data_list_converter/types/csv_file'
DataListConverter.convert(:item_data, :csv_file, data, csv_file: {filename: 'result.csv'})
DataListConverter.convert(:csv_file, :item_data, {filename: 'result.csv'}) == data

require 'data_list_converter/types/xls_file'
sheets_data = {sheet1: data, sheet2: data}
DataListConverter.convert(:multi_sheet_item_data, :xls_file, sheets_data, xls_file: {filename: 'result.xls'})
DataListConverter.convert(:xls_file, :multi_sheet_item_data, {filename: 'result.xls'}) == sheets_data
```

You can also add filter to this process:

```ruby
data = (1..20).map{|i| {name: "user-#{i}", age: i+20}}
# filter with default options (limit 10)
DataListConverter.convert(:item_data, :table_data, data, item_iterator: {filter: :limit})
# filter with options
DataListConverter.convert(:item_data, :table_data, data, item_iterator: {filter: {limit: {size: 2}}})
# multiple filters
DataListConverter.convert(:item_data, :table_data, data, item_iterator: {filter: [{limit: {size: 12}}, {count: {size: 4}}]})
```

## Data Types

Default data types:

- **item_data** like: `[{name: 'James', age: '22'}, ...]`, keys should be symbol.
- **item_iterator** iterator for item_data, used like: iter.call{|item| out << item}
- **table_data** like: `[["name", "age"], ["James", "22"], ["Bob", "33"], ...]`
- **table_iterator** iterator for table_data
- **multi_sheet** Contains several data with sheets:
    - **multi_sheet_table_iterator**: like: `{sheet1: table_iterator1, sheet2: table_iterator2}`
    - **multi_sheet_table_data**: like: `{sheet1: [['name', 'age'], ...], sheet2: ...}`
    - **multi_sheet_item_iterator**: like: `{sheet1: item_iterator1, sheet2: item_iterator2}`
    - **multi_sheet_item_data**: like: `{sheet1: [{name: 'James', age: 32}], sheet2: ...}`

Plugin data types, should required first by `require 'data_list_converter/types/#{type}'`

- **csv_file** file in csv format
- **xls_file** file in excel format, should install `spreadsheet` gem first
- **xlsx_file** file in excel xml format, should install `rubyXL` gem first
- **records** ActiveRecord records

Please check [test examples](https://github.com/halida/data_list_converter/blob/master/test/types_test.rb) to see how to use those types.

## Filters

**item_iterator/table_iterator limit**: limit item_iterator result counts, usage: `DataListConverter.convert(:item_data, :table_data, item_data, item_iterator: {filter: {limit: {size: 2}}})`, default limit size is 10.

**item_iterator count**: count item_iterator items, usage: `DataListConverter.convert(:xls_file, :item_data, {filename: 'result.xls'}, item_iterator: {filter: {count: {size: 10}}})`, it will print current item counts every `size`, please [see here](https://github.com/halida/data_list_converter/blob/master/lib/data_list_converter/filters/count.rb) for more options.

Please see [test examples](https://github.com/halida/data_list_converter/blob/master/test/filters_test.rb) to learn how to use filter.

## helpers

- **types**: Get current valid types.
- **routes**: Get current valid routes.
- **file_types**: Get current file types, which is the types has `_file` suffix.
- **get_file_format**: Get file type by filename, which compare file extension with `file_types`. `DataListConverter.get_file_format('xxx.xls') == :xls_file`
- **save_to_file**: Save data to file, it will use `get_file_format` to find proper file format.  `DataListConverter.save_to_file(filename, data, data_format=:item_data)`
- **load_from_file**: Get data from file, it will use `get_file_format` to find proper file format. `DataListConverter.load_from_file(filename, data_format=:item_data)`
- **unify_item_data_keys**: Sometimes in the `item_data` list, each data keys don't exactly same, so use this function to fix it, example: `DataListConverter.unify_item_data_keys([{a: 12}, {b: 11}]) == [{a: 12, b: nil}, {a: nil, b: 11}]`.
- **flatten**: Flatten multi level item_data list into one level, example: `DataListConverter.flatten({a: {b: 12}, c: {d: {e: 11}}}) == {:"a:b"=>12, :"c:d:e"=>11}`, can change seperator: `DataListConverter.flatten(data, '_')`, set max level: `DataListConverter.flatten(data, '_', 2)`

## Extend

You can add your own data types and filters, example:

```ruby
DataListConverter.register_converter(:records, :item_iterator) do |input, options|
  query = self.parameter(input, :query, :input)
  columns = self.parameter(input, :columns, :input)
  display = input[:display] || columns

  lambda { |&block|
    query.pluck(*columns).each do |data|
      item = {}
      data.each_with_index do |d, i|
        item[display[i]] = d
      end
      block.call(item)
    end
  }
end

DataListConverter.register_filter(:item_iterator, :limit) do |proc, options|
  limit_size = options[:size] || 10
  lambda { |&block|
    limit = 0
    proc.call do |item|
      block.call(item)
      limit += 1
      break if limit >= limit_size
    end
  }
end
```
