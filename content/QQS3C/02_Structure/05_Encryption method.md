---
title: 05. Encryption method
---
# Homomorphic encryption
## Intro
Homomorphic Encryption (HE) is a form of encryption that permits computations to be performed directly on encrypted data (ciphertexts) without requiring decryption first. The result of these computations remains encrypted, and when decrypted, it is identical to the result that would have been obtained by performing the same operations on the original, unencrypted data (plaintext). This technology enables data processing by untrusted third parties while maintaining strict data confidentiality.

---

**BGV (Brakerski-Gentry-Vaikuntanathan):** A leveled fully homomorphic encryption scheme that is highly regarded for its support of **integer arithmetic**. It excels at performing many additions and multiplications (polynomial operations) on packed ciphertexts using the **Single Instruction, Multiple Data (SIMD)** paradigm. Its noise management is typically handled via **modulus switching**.

**BFV (Brakerski/Fan-Vercauteren):** A leveled fully homomorphic encryption scheme also designed for **integer arithmetic**, similar to BGV. BFV and BGV are closely related and often used for similar tasks. A key difference lies in their noise management techniques; BFV generally manages noise by scaling the plaintext.

**CKKS (Cheon-Kim-Kim-Song):** A leveled homomorphic encryption scheme designed specifically for **real or complex number (fixed-point) arithmetic**. Unlike BGV and BFV, CKKS treats noise as part of the message, allowing for **approximate computations**. This makes it extremely efficient and well-suited for applications like machine learning and statistical analysis, where exact precision is not always required.

---

In this section, we will **exclusively use the BGV scheme**.
## RGSW
The **Go version** of the code utilizes a **special encryption scheme**. The description for this can be found at the following link: [CDSL-EncryptedControl/CDSL: Code for 2024 SICE documentation](https://github.com/CDSL-EncryptedControl/CDSL).