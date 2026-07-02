
# Mã nguồn Spring Boot

### 1. Entity (`Account.java`)
```java
package com.example.banking.entity;

import jakarta.persistence.*;
import lombok.*;

/**
 * Entity đại diện cho bảng lưu trữ thông tin Tài khoản cơ bản trong hệ thống.
 */
@Entity
@Table(name = "accounts")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Account {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String fullName;

    @Column(nullable = false, length = 15)
    private String phone;

    @Column(nullable = false, length = 100, unique = true)
    private String email;

    @Column(nullable = false, length = 12, unique = true)
    private String citizenId;

    @Column(nullable = false, unique = true)
    private String accountNumber;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private AccountStatus status;
}

enum AccountStatus {
    PENDING, ACTIVE
}
```

### 2. DTO (Data Transfer Objects)

**`AccountRegistrationRequest.java`**
```java
package com.example.banking.dto.request;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Pattern;
import lombok.Data;

/**
 * DTO nhận dữ liệu yêu cầu đăng ký mở tài khoản cơ bản từ Client.
 * Chứa các validation rule đảm bảo dữ liệu đầu vào hợp lệ.
 */
@Data
public class AccountRegistrationRequest {

    @NotBlank(message = "Họ và tên không được để trống")
    private String fullName;

    @NotBlank(message = "Số điện thoại không được để trống")
    @Pattern(regexp = "^(0|\\+84)[0-9]{9}$", message = "Số điện thoại không hợp lệ")
    private String phone;

    @NotBlank(message = "Email không được để trống")
    @Email(message = "Định dạng email không hợp lệ")
    private String email;

    @NotBlank(message = "Số CCCD không được để trống")
    @Pattern(regexp = "^[0-9]{12}$", message = "Số CCCD phải bao gồm đúng 12 chữ số")
    private String citizenId;
}
```

**`AccountRegistrationResponse.java`**
```java
package com.example.banking.dto.response;

import lombok.Builder;
import lombok.Data;

/**
 * DTO trả về dữ liệu cho Client sau khi đăng ký tài khoản thành công.
 */
@Data
@Builder
public class AccountRegistrationResponse {
    private Long accountId;
    private String accountNumber;
    private String status;
}
```

### 3. Repository (`AccountRepository.java`)
```java
package com.example.banking.repository;

import com.example.banking.entity.Account;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

/**
 * Repository interface để tương tác với cơ sở dữ liệu bảng accounts.
 */
@Repository
public interface AccountRepository extends JpaRepository<Account, Long> {
    
    // Kiểm tra xem số CCCD đã tồn tại trong hệ thống chưa
    boolean existsByCitizenId(String citizenId);
    
    // Kiểm tra xem email đã tồn tại trong hệ thống chưa
    boolean existsByEmail(String email);
}
```

### 4. Service

**`AccountService.java`** (Interface)
```java
package com.example.banking.service;

import com.example.banking.dto.request.AccountRegistrationRequest;
import com.example.banking.dto.response.AccountRegistrationResponse;

/**
 * Interface định nghĩa các nghiệp vụ liên quan đến Quản lý tài khoản.
 */
public interface AccountService {
    
    /**
     * Xử lý logic đăng ký mở tài khoản cơ bản.
     * 
     * @param request Dữ liệu đăng ký từ client
     * @return AccountRegistrationResponse Thông tin tài khoản sau khi tạo
     */
    AccountRegistrationResponse registerBasicAccount(AccountRegistrationRequest request);
}
```

**`AccountServiceImpl.java`** (Implementation)
```java
package com.example.banking.service.impl;

import com.example.banking.dto.request.AccountRegistrationRequest;
import com.example.banking.dto.response.AccountRegistrationResponse;
import com.example.banking.entity.Account;
import com.example.banking.entity.AccountStatus;
import com.example.banking.repository.AccountRepository;
import com.example.banking.service.AccountService;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Random;

/**
 * Lớp triển khai các logic nghiệp vụ cho AccountService.
 */
@Service
@RequiredArgsConstructor
public class AccountServiceImpl implements AccountService {

    private final AccountRepository accountRepository;

    @Override
    @Transactional
    public AccountRegistrationResponse registerBasicAccount(AccountRegistrationRequest request) {
        
        // 1. Kiểm tra tính duy nhất của CCCD và Email
        if (accountRepository.existsByCitizenId(request.getCitizenId())) {
            throw new IllegalArgumentException("Số CCCD đã tồn tại trong hệ thống.");
        }
        if (accountRepository.existsByEmail(request.getEmail())) {
            throw new IllegalArgumentException("Email đã được sử dụng.");
        }

        // 2. Sinh số tài khoản ngẫu nhiên (Giả lập logic sinh số tài khoản 10 số)
        String generatedAccountNumber = generateRandomAccountNumber();

        // 3. Mapping dữ liệu từ Request DTO sang Entity
        Account newAccount = Account.builder()
                .fullName(request.getFullName())
                .phone(request.getPhone())
                .email(request.getEmail())
                .citizenId(request.getCitizenId())
                .accountNumber(generatedAccountNumber)
                .status(AccountStatus.PENDING) // Mặc định trạng thái là PENDING chờ duyệt/kích hoạt
                .build();

        // 4. Lưu vào Database
        Account savedAccount = accountRepository.save(newAccount);

        // 5. Trả về Response DTO cho Controller
        return AccountRegistrationResponse.builder()
                .accountId(savedAccount.getId())
                .accountNumber(savedAccount.getAccountNumber())
                .status(savedAccount.getStatus().name())
                .build();
    }

    /**
     * Hàm phụ trợ (helper method) sinh số tài khoản ngẫu nhiên 10 chữ số.
     */
    private String generateRandomAccountNumber() {
        Random rnd = new Random();
        int number = rnd.nextInt(999999999) + 100000000;
        return "1" + String.format("%09d", number);
    }
}
```

### 5. Controller (`AccountController.java`)
```java
package com.example.banking.controller;

import com.example.banking.dto.request.AccountRegistrationRequest;
import com.example.banking.dto.response.AccountRegistrationResponse;
import com.example.banking.service.AccountService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

/**
 * Controller xử lý các HTTP requests liên quan đến Tài khoản.
 */
@RestController
@RequestMapping("/api/v1/accounts")
@RequiredArgsConstructor
public class AccountController {

    private final AccountService accountService;

    /**
     * API Endpoint cho phép client đăng ký mở tài khoản cơ bản.
     * 
     * @param request Dữ liệu từ body request đã được validate (@Valid)
     * @return ResponseEntity chứa thông tin tài khoản được tạo và HTTP status 200 OK
     */
    @PostMapping("/register")
    public ResponseEntity<AccountRegistrationResponse> registerAccount(
            @Valid @RequestBody AccountRegistrationRequest request) {
        
        // Gọi Service xử lý logic đăng ký
        AccountRegistrationResponse response = accountService.registerBasicAccount(request);
        
        // Trả kết quả về cho Client
        return ResponseEntity.ok(response);
    }
}
```
