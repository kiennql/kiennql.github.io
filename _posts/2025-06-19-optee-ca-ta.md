---
title: "OP-TEE: Phân tích quy trình gọi từ CA đến TA (Phần 2)"
author: kiennql
date: 2025-06-19 14:32:00 +0700
categories: [optee-labs]
tags: [op-tee, arm trustzone, call flow, debugging, ca, ta, smc, atf, secure world, analysis]
math: true
mermaid: true
render_with_liquid: false
---

## 1. Mục lục
- [1. Mục lục](#1-mục-lục)
- [2. Môi trường](#2-môi-trường)
- [3. Quy trình gọi (Call Flow) phân tích](#3-quy-trình-gọi-call-flow-phân-tích)
  - [3.1. Tổng quan](#31-tổng-quan)
  - [3.2. Quy trình làm việc của CA \& TA](#32-quy-trình-làm-việc-của-ca--ta)
- [4. Phân tích mã nguồn](#4-phân-tích-mã-nguồn)
  - [4.1. TEEC\_InitializeContext](#41-teec_initializecontext)
  - [4.2. TEEC\_OpenSession](#42-teec_opensession)
  - [4.3. TEEC\_InvokeCommand](#43-teec_invokecommand)
  - [4.4. TEEC\_CloseSession](#44-teec_closesession)
  - [4.5. TEEC\_FinalizeContext](#45-teec_finalizecontext)

Bài viết này phân tích chi tiết quy trình gọi từ Client Application (CA) đến Trusted Application (TA) trong OP-TEE.

## 2. Môi trường

- Ubuntu 22.04
- ARM Development Studio (ADS) + OP-TEE FVP

## 3. Quy trình gọi (Call Flow) phân tích

### 3.1. Tổng quan

Từ góc độ tổng quan, toàn bộ quy trình gọi từ CA đến TA:

```
CA → OP-TEE Client → TEE Driver → ATF → TEE → TA
```

![Call Flow Diagram](/assets/img/post/optee-call-flow/image.png)
_Sơ đồ tổng quan quy trình gọi từ CA đến TA_

### 3.2. Quy trình làm việc của CA & TA

**Client Application (CA):**

```c
// 1. Khởi tạo context để tương tác với TEE
res = TEEC_InitializeContext(NULL, &ctx);

// 2. Mở session, TEE sẽ xác thực và tải TA tương ứng
res = TEEC_OpenSession(&ctx, &sess, &uuid,
                       TEEC_LOGIN_PUBLIC, NULL, NULL, &err_origin);

// 3. Tương tác thông qua invoke command
res = TEEC_InvokeCommand(&sess, TA_HELLO_WORLD_CMD_INC_VALUE, &op,
                         &err_origin);

// 4. Đóng session
TEEC_CloseSession(&sess);

// 5. Giải phóng context
TEEC_FinalizeContext(&ctx);
```

**Trusted Application (TA):**

```c
// 1. Entry points khi TA được tải
TA_CreateEntryPoint
TA_OpenSessionEntryPoint

// 2. Xử lý business logic
TEE_Result TA_InvokeCommandEntryPoint(void __maybe_unused *sess_ctx,
                                      uint32_t cmd_id,
                                      uint32_t param_types, 
                                      TEE_Param params[4])
{
    switch (cmd_id) {
    case TA_HELLO_WORLD_CMD_INC_VALUE:
        return inc_value(param_types, params);
    case TA_HELLO_WORLD_CMD_DEC_VALUE:
        return dec_value(param_types, params);
    default:
        return TEE_ERROR_BAD_PARAMETERS;
    }
}

// 3. Cleanup khi đóng session
TA_CloseSessionEntryPoint
TA_DestroyEntryPoint
```

**Mối quan hệ CA-TA:**

```
TEEC_OpenSession    → TA_CreateEntryPoint + TA_OpenSessionEntryPoint
TEEC_InvokeCommand  → TA_InvokeCommandEntryPoint
TEEC_CloseSession   → TA_CloseSessionEntryPoint + TA_DestroyEntryPoint
```

## 4. Phân tích mã nguồn

### 4.1. TEEC_InitializeContext

Mở TEE driver và chuẩn bị giao tiếp:

```c
TEEC_InitializeContext
└── teec_open_dev
    └── ioctl(fd, TEE_IOC_VERSION, &vers)
```

Command được sử dụng: `TEE_IOC_VERSION`

### 4.2. TEEC_OpenSession

Quá trình mở session phức tạp nhất, bao gồm việc tải TA:

```c
TEEC_OpenSession(&ctx, &sess, &uuid, TEEC_LOGIN_PUBLIC, NULL, NULL, &err_origin);
└── ioctl(ctx->fd, TEE_IOC_OPEN_SESSION, &buf_data);
```

**Trong TEE Driver:**

```c
tee_ioctl_open_session
└── ctx->teedev->desc->ops->open_session(ctx, &arg, params);
    └── optee_open_session
        └── optee->ops->do_call_with_arg(ctx, shm_arg, offs);
```

**SMC Call đến ATF:**

```c
// linux/drivers/tee/optee/smc_abi.c
optee->smc.invoke_fn(param.a0, param.a1, param.a2, param.a3,
                     param.a4, param.a5, param.a6, param.a7, &res);
```

**Trong ATF:**
- SMC #0 interrupt → Exception Level 3
- `opteed_smc_handler` xử lý request
- Lưu non-secure context, phục hồi secure context
- ERET vào OP-TEE

**Trong OP-TEE:**

```c
thread_handle_std_smc
└── thread_alloc_and_run
    └── thread_std_smc_entry
        └── __tee_entry_std
            └── entry_open_session (OPTEE_MSG_CMD_OPEN_SESSION)
                └── tee_ta_open_session
                    └── tee_ta_init_session // Tải TA
                    └── ts_ctx->ops->enter_open_session
                        └── user_ta_enter_open_session
                            └── thread_enter_user_mode // S-EL1 → S-EL0
```

### 4.3. TEEC_InvokeCommand

Logic tương tự OpenSession, khác biệt ở command:

```c
TEEC_InvokeCommand
└── ioctl(TEE_IOC_INVOKE_COMMAND)
    └── entry_invoke_command (OPTEE_MSG_CMD_INVOKE_COMMAND)
        └── user_ta_enter_invoke_cmd
            └── user_ta_enter(UTEE_ENTRY_FUNC_INVOKE_COMMAND)
```

Cuối cùng gọi đến `TA_InvokeCommandEntryPoint` trong TA.

### 4.4. TEEC_CloseSession

Đóng session và cleanup:

```c
void TEEC_CloseSession(TEEC_Session *session)
{
    struct tee_ioctl_close_session_arg arg;
    
    arg.session = session->session_id;
    ioctl(session->ctx->fd, TEE_IOC_CLOSE_SESSION, &arg);
}
```

Command: `TEE_IOC_CLOSE_SESSION` → `TA_CloseSessionEntryPoint`

### 4.5. TEEC_FinalizeContext

Đóng driver:

```c
void TEEC_FinalizeContext(TEEC_Context *ctx)
{
    if (ctx)
        close(ctx->fd);
}
```

**Chúc mừng!** Bạn đã hiểu được quy trình hoàn chỉnh từ CA đến TA trong OP-TEE ecosystem. Kiến thức này là nền tảng quan trọng để phát triển và debug các ứng dụng bảo mật trên ARM TrustZone.
