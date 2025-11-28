# Employee Search Service - Spring Boot &amp

ElasticsearchThis repository contains a Spring Boot application that connects to an Elasticsearch cluster. It processes search requests involving interest and salary criteria, converts them into Elasticsearch queries, and returns matching employee documents.

## Project StructureLanguage: 
Java 17Framework: Spring Boot 3.2.0Database:
ElasticsearchBuild Tool: Maven
## 1. Configuration (pom.xml)Required dependencies for Spring Web and Spring Data Elasticsearch.&l
```
<;?xml version="1.0" encoding="UTF-8"?&gt;
&lt;project xmlns="[http://maven.apache.org/POM/4.0.0](http://maven.apache.org/POM/4.0.0)" 
         xmlns:xsi="[http://www.w3.org/2001/XMLSchema-instance](http://www.w3.org/2001/XMLSchema-instance)"
         xsi:schemaLocation="[http://maven.apache.org/POM/4.0.0](http://maven.apache.org/POM/4.0.0) [https://maven.apache.org/xsd/maven-4.0.0.xsd](https://maven.apache.org/xsd/maven-4.0.0.xsd)"&gt;
    &lt;modelVersion&gt;4.0.0&lt;/modelVersion&gt;
    &lt;parent&gt;
        &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
        &lt;artifactId&gt;spring-boot-starter-parent&lt;/artifactId&gt;
        &lt;version&gt;3.2.0&lt;/version&gt;
        &lt;relativePath/&gt; 
    &lt;/parent&gt;
    &lt;groupId&gt;com.example&lt;/groupId&gt;
    &lt;artifactId&gt;es-search-service&lt;/artifactId&gt;
    &lt;version&gt;0.0.1-SNAPSHOT&lt;/version&gt;
    &lt;name&gt;es-search-service&lt;/name&gt;
    &lt;description&gt;Elasticsearch Search Service&lt;/description&gt;

    &lt;properties&gt;
        &lt;java.version&gt;17&lt;/java.version&gt;
    &lt;/properties&gt;

    &lt;dependencies&gt;
        &lt;dependency&gt;
            &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
            &lt;artifactId&gt;spring-boot-starter-data-elasticsearch&lt;/artifactId&gt;
        &lt;/dependency&gt;
        &lt;dependency&gt;
            &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
            &lt;artifactId&gt;spring-boot-starter-web&lt;/artifactId&gt;
        &lt;/dependency&gt;
        &lt;dependency&gt;
            &lt;groupId&gt;org.projectlombok&lt;/groupId&gt;
            &lt;artifactId&gt;lombok&lt;/artifactId&gt;
            &lt;optional&gt;true&lt;/optional&gt;
        &lt;/dependency&gt;
    &lt;/dependencies&gt;

    &lt;build&gt;
        &lt;plugins&gt;
            &lt;plugin&gt;
                &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
                &lt;artifactId&gt;spring-boot-maven-plugin&lt;/artifactId&gt;
            &lt;/plugin&gt;
        &lt;/plugins&gt;
    &lt;/build&gt;
&lt;/project&gt;
## 2. Source Code### Employee EntityFile: src/main/java/com/example/search/Employee.javaMaps the Elasticsearch document structure.package com.example.search;

import lombok.Data;
import org.springframework.data.annotation.Id;
import org.springframework.data.elasticsearch.annotations.Document;
import org.springframework.data.elasticsearch.annotations.Field;
import org.springframework.data.elasticsearch.annotations.FieldType;
import java.util.List;

@Data
@Document(indexName = "employees")
public class Employee {
    @Id
    private String id;

    @Field(type = FieldType.Text, name = "firstName")
    private String firstName;

    @Field(type = FieldType.Text, name = "lastName")
    private String lastName;

    @Field(type = FieldType.Keyword, name = "designation")
    private String designation;

    @Field(type = FieldType.Long, name = "salary")
    private Long salary;

    @Field(type = FieldType.Date, name = "dateOfJoining")
    private String dateOfJoining;

    @Field(type = FieldType.Text, name = "address")
    private String address;

    @Field(type = FieldType.Keyword, name = "gender")
    private String gender;

    @Field(type = FieldType.Integer, name = "age")
    private Integer age;

    @Field(type = FieldType.Keyword, name = "maritalStatus")
    private String maritalStatus;

    @Field(type = FieldType.Text, name = "interests")
    private List&lt;String&gt; interests;
}
### Request/Response ModelsFile: src/main/java/com/example/search/SearchModels.javaDTOs for the REST API.package com.example.search;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import java.util.List;

public class SearchModels {

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    public static class SearchRequest {
        private String interest;
        private Long minSalary;
    }

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    public static class SearchResponse {
        private long count;
        private List&lt;Employee&gt; results;
    }
}


          
            
          
        
  
        
    

Search Service LogicFile: src/main/java/com/example/search/SearchService.javaConstructs the native Elasticsearch query using NativeQuery and Query DSL.package com.example.search;

import co.elastic.clients.elasticsearch._types.query_dsl.Query;
import com.example.search.SearchModels.SearchRequest;
import com.example.search.SearchModels.SearchResponse;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.elasticsearch.client.elc.NativeQuery;
import org.springframework.data.elasticsearch.core.ElasticsearchOperations;
import org.springframework.data.elasticsearch.core.SearchHits;
import org.springframework.data.elasticsearch.core.SearchHit;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.stream.Collectors;

@Service
public class SearchService {

    private final ElasticsearchOperations elasticsearchOperations;

    public SearchService(ElasticsearchOperations elasticsearchOperations) {
        this.elasticsearchOperations = elasticsearchOperations;
    }

    public SearchResponse search(SearchRequest request) {
        // 1. Match interest
        Query interestQuery = Query.of(q -&gt; q
            .match(m -&gt; m.field("interests").query(request.getInterest()))
        );

        // 2. Filter by Salary
        Query salaryQuery = Query.of(q -&gt; q
            .range(r -&gt; r.field("salary").gte(co.elastic.clients.json.JsonData.of(request.getMinSalary())))
        );

        // 3. Combine (Bool Query)
        Query boolQuery = Query.of(q -&gt; q
            .bool(b -&gt; b.must(interestQuery).filter(salaryQuery))
        );

        NativeQuery nativeQuery = NativeQuery.builder()
                .withQuery(boolQuery)
                .withPageable(PageRequest.of(0, 10))
                .build();

        SearchHits&lt;Employee&gt; searchHits = elasticsearchOperations.search(nativeQuery, Employee.class);

        List&lt;Employee&gt; employees = searchHits.stream()
                .map(SearchHit::getContent)
                .collect(Collectors.toList());

        return new SearchResponse(searchHits.getTotalHits(), employees);
    }
}
```
### REST ControllerFile: 
src/main/java/com/example/search/SearchController.javapackage com.example.search;
```
import com.example.search.SearchModels.SearchRequest;
import com.example.search.SearchModels.SearchResponse;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/search")
public class SearchController {

    private final SearchService searchService;

    public SearchController(SearchService searchService) {
        this.searchService = searchService;
    }

    @PostMapping
    public ResponseEntity&lt;SearchResponse&gt; searchEmployees(@RequestBody SearchRequest request) {
        if (request.getInterest() == null || request.getMinSalary() == null) {
            return ResponseEntity.badRequest().build();
        }
        return ResponseEntity.ok(searchService.search(request));
    }
}
```
### Application Entry PointFile:

src/main/java/com/example/search/SearchApplication.javapackage com.example.search;
```
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SearchApplication {
    public static void main(String[] args) {
        SpringApplication.run(SearchApplication.class, args);
    }
}
```
## 3. How to RunStart Elasticsearch: Ensure Elasticsearch is running locally on port 9200.

```
Build: mvn clean installRun: mvn spring-boot:run## 4. API UsageEndpoint: POST /searchRequest Body:{
    "interest": "Aircraft Spotting",
    "minSalary": 70000
}
Response:{
    "count": 93,
    "results": [
        {
            "firstName": "LIA",
            "lastName": "BELICH",
            "designation": "Manager",
            "salary": 84000,
            "interests": ["Aircraft Spotting"]
            // ... other fields
        }
    ]
}
