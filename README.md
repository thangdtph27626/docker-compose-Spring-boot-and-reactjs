# Demo full quy trình hoạt động của một dự án khi sử dụng docker

Trong bài viết này chúng ta sử dụng docker compose để cấu hình 3 container: 
   + Container cơ sở dữ liệu( Postgres )
   + Container BE (Spring boot rest full API)
   + Container FE (Reactjs)

## Khởi tạo project res full API

### 1: Khởi tạo một project apring boot bằng intellj
### 2: Cấu hình thư mục dự án 

![image](https://github.com/thangdtph27626/docker-compose-Spring-boot-and-reactjs/assets/109157942/26699ddd-da15-4565-bd9e-ea17a901161c)

### 3:Thiết lập dự án

config trong file application.properties

```
spring.datasource.url=${OPENSHIFT_POSTGRESQL_DB_HOST}:${OPENSHIFT_POSTGRESQL_DB_PORT}/${OPENSHIFT_POSTGRESQL_DB_DATABASE}
spring.datasource.username=${OPENSHIFT_POSTGRESQL_DB_USER_NAME}
spring.datasource.password=${OPENSHIFT_POSTGRESQL_DB_PASSWORD}
spring.jpa.properties.hibernate.jdbc.lob.non_contextual_creation= true
spring.jpa.properties.hibernate.dialect= org.hibernate.dialect.PostgreSQLDialect
spring.jpa.hibernate.ddl-auto= update
spring.security.oauth2.client.registration.google.client-id=${OPENSHIFT_OAUTH2_CLIENT_ID}
spring.security.oauth2.client.registration.google.client-secret=${OPENSHIFT_OAUTH2_CLIENT_SECRET}
client.id.google=${OPENSHIFT_OAUTH2_CLIENT_ID}
jwt.secret=${JWT_SECRET}
```

### 4: tạo lớp thực thể (Entity Class)

```
package com.example.demo.model;

import lombok.Data;
import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;
import javax.persistence.Table;


@Data
@Entity
@Table(name = "sinh_vien")
public class SinhVien {

    @Id
    @Column(name = "ma_sinh_vien")
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long maSinhVien;

    @Column(name = "ten_sinh_vien", nullable = true)
    private String tenSinhVien;
}
```

### 5: Tạo Spring Data JPA Repository

Spring Data JPA API cung cấp hỗ trợ kho lưu trữ cho Java Persistence API (JPA) và nó giúp giảm bớt sự phát triển của các ứng dụng cần truy cập các nguồn dữ liệu JPA.
Tôi sẽ tạo giao diện kho lưu trữ và bạn không cần tạo bất kỳ phương thức nào trong giao diện này vì Spring cung cấp các phương thức để thực hiện các thao tác CRUD cơ bản.

```

package com.example.demo.repository;

import com.example.demo.model.SinhVien;
import org.springframework.data.jpa.repository.JpaRepository;

public interface SinhVienRepo extends JpaRepository<SinhVien, Long> {
}
```

### 6: Tạo lớp Service
Lớp Service nằm giữa bộ controller và lớp DAO và chứa các phương thức trừu tượng. Nói chung, bạn thực hiện gọi các hàm trong lớp dịch vụ này.
```markdown
package com.example.demo.service;

import com.example.demo.model.SinhVien;
import com.example.demo.request.SinhVienRequest;

import java.util.List;

public interface SinhVienService {

     List<SinhVien> getList();

     SinhVien addNew(SinhVienRequest sinhVien);

     boolean delete(long id);

     SinhVien update(long id, SinhVienRequest sinhVien);

     SinhVien findById(long id);

}
```
### 7: implements

Lớp  **SinhVienServiceImpl** ghi đè các phương thức của  **SinhVienService**  và thực hiện logic nghiệp vụ trong lớp dịch vụ này\
Bạn nhận được kết quả của các truy vấn nối từ kho lưu trữ và chuyển cho lớp điều khiển REST.\
Tôi sử dụng cùng một phương pháp để lưu hoặc cập nhật thông tin công ty mới hoặc hiện có tương ứng.

```markdown
package com.example.demo.service.impl;

import com.example.demo.model.SinhVien;
import com.example.demo.repository.SinhVienRepo;
import com.example.demo.request.SinhVienRequest;
import com.example.demo.service.SinhVienService;
import org.springframework.beans.BeanUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Optional;

@Service
public class SinhVienServiceImpl implements SinhVienService {

    @Autowired
    private SinhVienRepo sinhVienRepository;

    @Override
    public List<SinhVien> getList() {
        return sinhVienRepository.findAll();
    }

    @Override
    public SinhVien addNew(SinhVienRequest sinhVienRequest) {
        SinhVien sinhVien = new SinhVien();
        BeanUtils.copyProperties(sinhVienRequest, sinhVien);
        return sinhVienRepository.save(sinhVien);
    }

    @Override
    public boolean delete(long id) {
        Optional<SinhVien> sinhVien = sinhVienRepository.findById(id);
        if(sinhVien.isPresent()){
            sinhVienRepository.deleteById(id);
            return true;
        }
        return false;
    }

    @Override
    public SinhVien update(long id, SinhVienRequest sinhVienRequest) {
        SinhVien sinhVien = sinhVienRepository.findById(id).orElse(null);
        sinhVien.setTenSinhVien(sinhVienRequest.getTenSinhVien());
        return sinhVienRepository.save(sinhVien);
    }

    @Override
    public SinhVien findById(long id) {
        return sinhVienRepository.findById(id).orElse(null);
    }
}
```

### 8: RestController

```
package com.example.demo.controller;

import com.example.demo.model.SinhVien;
import com.example.demo.request.SinhVienRequest;
import com.example.demo.service.SinhVienService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@RestController
@RequestMapping("/api")
public class SinhVienResController {

    @Autowired()
    private SinhVienService sinhVienService;

    @PostMapping()
    public SinhVien addNew(@RequestBody SinhVienRequest sinhVien){
        System.out.println(sinhVien.getTenSinhVien());
        return sinhVienService.addNew(sinhVien);
    }

    @GetMapping("/{id}")
    public SinhVien DetailSinhVien(@PathVariable("id") long id){
        SinhVien sinhVien = null;
        try {
            sinhVien = sinhVienService.findById(id);
        }catch (Exception e){
            System.out.println(e);
        }
        return sinhVien;
    }

    @PutMapping("/{id}")
    public SinhVien update(@PathVariable("id") long id, @RequestBody SinhVienRequest sinhVienRequest) {
        return sinhVienService.update(id, sinhVienRequest);
    }

    @DeleteMapping("/{id}")
    public boolean delete(@PathVariable("id") long id){
        return sinhVienService.delete(id);
    }

    @GetMapping()
    public List<SinhVien> listSinhVien(Model model){
        List<SinhVien> list = sinhVienService.getList();
        return list;
    }
}
```

## Tạo Dockerfile cho Spring Boot 

 config file pom như sau:

```
<build>
        <plugins>
            <plugin>
                <!-- Build an executable JAR -->
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <version>3.1.0</version>
                <configuration>
                    <archive>
                        <manifest>
                            <addClasspath>true</addClasspath>
                            <classpathPrefix>lib/</classpathPrefix>
                            <mainClass>com.example.shose.server.ServerApplication</mainClass>
                        </manifest>
                    </archive>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>16</source>
                    <target>16</target>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>15</source>
                    <target>15</target>
                </configuration>
            </plugin>
        </plugins>

    </build>
```

Sau khi config xong bạn build project sau khi buil thì project sẽ tạo một file .jar trong build/libs/...

Tạo một Dockerfile như sau 

```
FROM openjdk:17-jdk-alpine
EXPOSE 8080
ARG JAR_FILE=build/libs/project-0.0.1-SNAPSHOT-plain.jar
ADD ${JAR_FILE} ./manager-user.jar
ENTRYPOINT ["java","-jar","./manager-user.jar"]
```

## Tạo Dockerfile cho Reactjs 

Dự án React có thể được xây dựng bằng cách yarn build hoặc npm run build. Nó nên được triển khai trên máy chủ nginx. Vì vậy chúng ta có thể sử dụng ngnix làm base image. Chúng ta có thể định cấu hình nginx với cấu hình sau:

> nginx.conf

```
server {

  listen 80;

  location / {
    root   /usr/share/nginx/html;
    index  index.html index.htm;
    try_files $uri $uri/ /index.html;
  }

  location /api/ {
      proxy_pass http://backend:8080;
      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection 'upgrade';
      proxy_set_header Host $host;
      proxy_cache_bypass $http_upgrade;
   }

  error_page   500 502 503 504  /50x.html;

  location = /50x.html {
    root   /usr/share/nginx/html;
  }

}
```

Các đường dẫn bắt đầu bằng / sẽ được chuyển hướng đến đường dẫn nginx và nếu nó bắt đầu bằng /api/ thì nó sẽ được chuyển hướng đến phần phụ trợ. Chúng ta cần có cấu hình này vì cả backend và frontend đều chạy trên cùng một máy chủ. Bây giờ chúng ta có thể sao chép thư mục build vào thư mục nginx và cấu hình vào thư mục /etc/nginx/.

```
FROM nginx:1.16.0-alpine
COPY --from=build /app/build /usr/share/nginx/html
RUN rm /etc/nginx/conf.d/default.conf
COPY ./nginx/nginx.conf /etc/nginx/conf.d
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Chúng ta có thể tiến thêm một bước nữa và xây dựng dự án trong vùng chứa và chúng ta có thể thực hiện theo 2 bước trong Dockerfile:

```
FROM node:12.4.0-alpine as build
WORKDIR /app
ENV PATH /app/node_modules/.bin:$PATH
COPY package.json /app/package.json
RUN npm install --silent
RUN npm install react-scripts@3.0.1 -g --silent
COPY . /app
RUN npm run build

FROM nginx:1.16.0-alpine
COPY --from=build /app/build /usr/share/nginx/html
RUN rm /etc/nginx/conf.d/default.conf
COPY ./nginx/nginx.conf /etc/nginx/conf.d
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Tạo Docker-compose file

```
version: '3.9'
services:
  db:
    image: postgres:13
    container_name: "postgres"
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=123456789
      - POSTGRES_DB=demo
    ports:
      - "5432:5432"
    volumes:
      - ./db-data:/var/lib/postgresql/data

  backend:
    image: "thang/backend"
    container_name: "backend"
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://manager_user_db:5432/demo
      - SPRING_DATASOURCE_USERNAME=postgres
      - SPRING_DATASOURCE_PASSWORD=123456789
    ports:
      - "8080:8080"
    depends_on:
      - db

  frontend:
    image: "thang/frontend"
    container_name: "frontend"
    ports:
      - "80:80"
    depends_on:
      - backend
```

Lưu ý rằng tất cả các vùng chứa đều hoạt động trong cùng một mạng và chúng có thể giao tiếp với nhau bằng cách sử dụng tên dịch vụ như db, backend hoặc frontend. Ví dụ: trong chuỗi kết nối cơ sở dữ liệu ứng dụng mùa xuân phải là jdbc:postgresql://db:5432/postgres. Và ứng dụng React có thể thực hiện các lệnh gọi còn lại tới http://backend/api/* . Bây giờ chúng ta có thể chạy tất cả các dịch vụ:

```
docker-compose up

```
