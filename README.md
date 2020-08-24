# XmlDataExtractor

This gem provides a DSL for extracting formatted data from any XML structure.

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'xml_data_extractor'
```

And then execute:

    $ bundle install

Or install it yourself as:

    $ gem install xml_data_extractor

## Usage

The general ideia is to declare a ruby Hash that represents the fields structure, containing instructions of how every piece of data should be retrieved from the XML document.

```ruby
structure = { schemas: { character: { path: "xml/FirstName" } } }
xml = "<xml><FirstName>Gandalf</FirstName></xml>"

result = XmlDataExtractor.new(structure).parse(xml)

# result -> { character: "Gandalf" }
```

For convenience, you can write the structure in yaml, which can be easily converted to a ruby hash using `YAML.load(yml).deep_symbolize_keys`.

Considering the following yaml and xml:

```yml
schemas:
  description:
    path: xml/desc
    modifier: downcase
  amount:  
    path: xml/info/price
    modifier: to_f
```
```xml
<xml>
  <desc>HELLO WORLD</desc>
  <info>
    <price>123</price>
  </info>  
</xml>
```

The output is:
```ruby
{
  description: "hello world",
  amount: 123.0
}
```

### Defining the structure

The structure should be defined as a hash inside the `schemas` key. See the [complete example](https://github.com/monde-sistemas/xml_data_extractor/blob/master/spec/examples/complete_example_spec.rb).

When defining the structure you can combine any available command in order to extract and format the data as needed.

The available commands are separated in two general pusposes:

- [Navigation & Extraction](#navigation--extraction)
- [Formatting](#formatting)

### Navigation & Extraction:

The data extraction process is based on `Xpath` using Nokogiri.
* [Xpath introduction](https://blog.scrapinghub.com/2016/10/27/an-introduction-to-xpath-with-examples)  
* [Xpath cheatsheet](https://devhints.io/xpath)

#### path

Defines the `xpath` of the element.
The `path` is the default command of a field definition, so this:
```yml
  schemas:
    description: 
      path: xml/desc
```    
Is equivalent to this:
```yml
  schemas:
    description: xml/desc
```    

It can be defined as a string:
```yml
schemas:
  description:
    path: xml/some_field
```
```xml
<xml>
  <some_field>ABC</some_field>
</xml>
```
```ruby
{ description: "ABC" }
```

Or as a string array:
```yml
schemas:
  address:
    path: [street, info/city]     
```
```xml
<xml>
  <street>Diagon Alley</street>
  <info>
    <city>London</city>
  </info>  
</xml>
```
```ruby
{ address: ["Diagon Alley", "London"] }
```

And even as a hash array, for complex operations:
```yml
schemas:
  address:
    path:
      - path: street
        modifier: downcase
      - path: info/city
        modifier: upcase   
```
```ruby
{ address: ["diagon alley", "LONDON"] }
```

#### attr

Defines a tag attribute which the value should be extracted from, instead of the tag value itself:
```yml
schemas:
  description:
    path: xml/info 
    attr: desc
```
```xml
<xml>
  <info desc="ABC">some stuff<info>
</xml> 
```
```ruby
{ description: "ABC" }
```

Like the path, it can also be defined as a string array.

#### within

To define a root path for the fields:
```yml
schemas:  
  movie:
    within: info/movie_data
    title: original_title
    actor: main_actor

```
```xml
<xml>
  <info>
    <movie_data>
      <original_title>The Irishman</original_title>
      <main_actor>Robert De Niro</main_actor>
    </movie_data>
  </info>
</xml>
```
```ruby
{ movie: { title: "The Irishman", actor: "Robert De Niro" } }
```

#### array_of

Defines the path to a XML collection, which will be looped generating an array of hashes:
```yml
schemas:
  people:
    array_of: characters/character
    name: firstname
    age: age
```
```xml
<xml>
  <characters>
    <character>
      <firstname>Geralt</firstname>
      <age>97</age>
    </character>
    <character>
      <firstname>Yennefer</firstname>
      <age>102</age>
    </character>
  </characters>
</xml>
```
```ruby
{
  people: [
    { name: "Geralt", age: "97" },
    { name: "Yennefer", age: "102" }
  ]
}
```

If you need to loop trough nested collections, you can define an array of paths:
```yml
schemas:
  show:    
    within: show_data
    title: description
    people:
      array_of: [characters/character, info]
      name: name
```
```xml
<xml>
  <show_data>
    <description>Peaky Blinders</description>
    <characters>
      <character>
        <info>
          <name>Tommy Shelby</name>          
        </info>
      </character>
      <character>
        <info>
          <name>Arthur Shelby</name>          
        </info>
        <info>
          <name>Alfie Solomons</name>
        </info>
      </character>
    </characters>
  </show_data>
</xml>
```
```ruby
{
  show: {
    title: "Peaky Blinders",
    people: [
      { name: "Tommy Shelby" },
      { name: "Arthur Shelby" },
      { name: "Alfie Solomons" }      
    ]
  }  
}
```

### Formatting:

#### fixed

Defines a fixed value for the field:
```yml
  currency:
    fixed: BRL
```
```ruby
  { currency: "BRL" }
```

#### mapper

Uses a hash of predefined values to replace the extracted value with its respective option.
If the extracted value is not found in any of the mapper options, it will be replaced by the `default` value, but if the default value is not defined, the returned value is not replaced.
```yml
mappers:
  currencies:
    default: unknown
    options:      
      BRL: R$
      USD: [US$, $]
schemas:
  money:    
    array_of: curr_types/type
    path: symbol
    mapper: currencies
```
```xml
  <xml>
    <curr_type>
      <type>
        <symbol>US$</symbol>
      </type>
      <type>
        <symbol>R$</symbol>
      </type>
      <type>
        <symbol>RB</symbol>
      </type>      
      <type>
        <symbol>$</symbol>
      </type>      
    </curr_type>  
  </xml>
```
```ruby
  {
    money: ["USD", "BRL", "unknown", "USD"]
  }
```

#### modifier

Defines a method to be called on the returned value.
```yml
schemas:
  name:
    path: some_field
    modifier: upcase
```
```xml
<xml>
  <some_field>Lewandovski</some_field>
</xml>
```
```ruby
{ name: "LEWANDOVSKI" }
```

You can also pass parameters to the method. In this case you will have to declare the modifier as an array of hashes, with the `name` and `params` keys:
```yml
schemas:
  name:
    path: [firstname, lastname]
    modifier: 
      - name: join
        params: [" "]    
      - downcase      
```
```xml
<xml>
  <firstname>Robert</firstname>
  <lastname>Martin</lastname>
</xml>
```
```ruby
{ name: "robert martin" }
```

If you need to use custom methods, you can pass an object containing the methods in the initialization. The custom method will receive the value as parameter:
```yml
schemas:
  name:
    path: final_price
    modifier: format_as_float      
```
```xml
<xml>
  <final_price>R$ 12.99</final_price>  
</xml>
```
```ruby
class MyMethods 
  def format_as_float(value)
    value.gsub(/[^\d.]/, "").to_f    
  end
end

XmlDataExtractor.new(yml, MyMethods.new).parse(xml)
```
```ruby
{ price: 12.99 }
```
