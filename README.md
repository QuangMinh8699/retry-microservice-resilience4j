 ## Xử lý khi một service bị lỗi không nhận được message trong kiến trúc microservice
 #### Chúng ta sẽ sử dụng thư viện Resilience4j

 ```java
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
```
#### Các dependency cần thêm
```java
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

Retry là cách đơn giản và dễ tiếp cận nhất để xử lý các trường hợp service không nhận được phản hồi sau khi gửi đi một message
 Chúng ta cần lưu ý đến 2 điều chính trong retry:
```
- Retry tối đa bao nhiêu lần ?
- Mỗi lần retry phải đợi trong bao nhiêu giây
```
Cấu hình Retry trong file application.yml:
```yml
resilience4j:  
	retry:  
		instances:  
			#serviceA: Tên của method mà chúng ta cần retry
			serviceA:  
				#Max-attempts: số lần retry tối đa
				#Nếu retry quá số lần tối đa thì sẽ bắn ra exception
				max-attempts: 5  
				#Wait-duration: số lần mỗi retry phải đợi
				#Khi hết thời gian, sẽ chuyển sang retry mới
				wait-duration: 5s
```

Controller của serviceA:
```java
@RestController  
@RequestMapping("/a")  
public class ServiceAController {

	@Autowired  
	private RestTemplate restTemplate;  

	int count = 1;

	@GetMapping  
	@Retry(
		name = "serviceA",
		//serviceA chúng ta đã gọi ở trên file .yml  
		fallbackMethod = "serviceAFallback"
		//khi retry quá số lần tối đa,
		//method "serviceAFallback" được sử dụng
	)
	public String serviceA() {  
	  String url = "http://localhost:8081/b";
	  System.out.println(
		  "Retry " + count++ + " times at " + new Date()
	  );  
	  return restTemplate.getForObject(  
		    url,  
	            String.class  
	  );  
	}

	public String serviceAFallback(Exception e) {  
	  return "This is a fallback method for Service A";  
	}
	
}
```

Controller của serviceB:
```java
@RestController  
@RequestMapping("/b")  
public class ServiceBController {  

    @GetMapping  
    public String serviceB() {  
	  return "Service B is called from service A";  
    }  
  
}
```
#### Trường hợp thứ nhất:
#### Khi chúng ta chạy serviceA mà chưa chạy serviceB
console sẽ hiện ra như sau:
```java
Retry method called 1 times atMon Nov 21 11:11:46 ICT 2022
Retry method called 2 times atMon Nov 21 11:11:51 ICT 2022
Retry method called 3 times atMon Nov 21 11:11:56 ICT 2022
Retry method called 4 times atMon Nov 21 11:12:01 ICT 2022
Retry method called 5 times atMon Nov 21 11:12:06 ICT 2022
```
Như vậy, ta thấy được cứ mỗi 5s, method sẽ được retry lại một lần,
kèm với đó, website sẽ trả về:
```
This is a fallback method for Service A
```

#### Trường hợp thứ hai:
#### Khi chúng ta chạy serviceB rồi mới chạy serviceA
console sẽ hiện ra như sau:
```java
Retry method called 1 times atMon Nov 21 11:19:44 ICT 2022
```
Khi thành công, retry sẽ lập tức dừng lại,
website sẽ trả về:
```
Service B is called from service A
```
