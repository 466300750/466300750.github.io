---
layout: post
title: spring security 添加验证码
date: 2018-05-25 14:48:31
categories: spring
share: y
excerpt_separator: <!--more-->
---

一般情况下网站提供的用户登录的界面中除了让用户输入用户名和密码还需要输入随机验证码，来进行人机校验。本文采用spring security实现的思路.首先介绍spring security如何实现登录验证需求，然后介绍如何加入图片验证码。

<!--more-->

### Spring Security的工作方式
加入图片验证码

由于生产环境有多态机器，为了保证服务器的完全无状态，选择服务器生成一个precode，并持久化。

```
@RestController
public class OperationVerificationCodeController {
    private DefaultKaptcha defaultKaptcha;
    private OperationVerificationCodeService applicationService;

    public OperationVerificationCodeController(DefaultKaptcha defaultKaptcha, OperationVerificationCodeService applicationService) {
        this.defaultKaptcha = defaultKaptcha;
        this.applicationService = applicationService;
    }

    @GetMapping("/verification/preCode")
    public String preCode() {
        return UUID.randomUUID().toString();
    }

    @GetMapping("/verification/code")
    public void defaultKaptcha(@RequestParam("preCode") String preCode, HttpServletResponse response) throws IOException {
        String text = defaultKaptcha.createText();
        applicationService.save(OperationVerificationCode.of(preCode, text));
        BufferedImage image = defaultKaptcha.createImage(text);
        responseVerificationCodeImage(response, image);
    }

    private void responseVerificationCodeImage(HttpServletResponse response, BufferedImage image) throws IOException {
        response.setHeader("Cache-Control", "no-store");
        response.setHeader("Pragma", "no-cache");
        response.setDateHeader("Expires", 0);
        response.setContentType("image/jpeg");
        ServletOutputStream outputStream = null;
        try {
            outputStream = response.getOutputStream();
            ImageIO.write(image, "jpeg", outputStream);
            outputStream.flush();
        } finally {
            IOUtils.closeQuietly(outputStream);
        }
    }
}
```

```
@Service
public class OperationVerificationCodeService {

    private static final int DURATION_MINUTES = 5;

    private OperationVerificationCodeRepository operationVerificationCodeRepository;

    public OperationVerificationCodeService(OperationVerificationCodeRepository operationVerificationCodeRepository) {
        this.operationVerificationCodeRepository = operationVerificationCodeRepository;
    }

    public void save(OperationVerificationCode operationVerificationCode) {
        operationVerificationCodeRepository.save(operationVerificationCode);
    }

    public void verify(String preCode, String verificationCode) {
        OperationVerificationCode code = operationVerificationCodeRepository.by(preCode);
        try {
            if (Objects.isNull(code) || !code.getVerificationCode().equalsIgnoreCase(verificationCode)) {
                throw new AccessDeniedException("验证码填写错误");
            }
            if (code.getCreatedAt().plus(DURATION_MINUTES, MINUTES).isBefore(Instant.now())) {
                throw new AuthenticationServiceException("验证码过期");
            }
        } finally {
            operationVerificationCodeRepository.delete(code.getId());
        }
    }
}
```

