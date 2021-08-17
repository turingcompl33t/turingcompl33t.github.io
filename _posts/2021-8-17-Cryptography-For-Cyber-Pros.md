---
layout: "post"
title: "Cryptography for Cybersecurity Professionals"
---

This post links a presentation I made back in 2019 titled "Cryptography for Cybersecurity Professionals." I lost track of the presentation in the intervening years, but recently stumbled on it again while auditing my password manager for old, unused accounts. 

As the name suggests, at the time I made the presentation, the intended audience was a group of cybersecurity students, meaning that it assumes some general technical competence but no prior background in cryptography.

While the title denotes the intended audience, it does little to foreshadow the actual content of the presentation. What is it that cybersecurity professionals really need to know about cryptography? There probably isn't general consensus on the answer to this question. A more appropriate title would be "How Cryptography Breaks" or something similar; the presentation focuses on the gulf between "theory" and "practice" in cryptography, and how this distinction renders invalid many of our assumptions about the strength of cryptography in the real world.

This presentation is heavily inspired by content originally by Professor Matthew Green of Johns Hopkins University, particularly two presentations: [one from 2013](https://www.youtube.com/watch?v=uP6np_oKVCk&t=3082s) delivered at my alma mater [and another](https://www.youtube.com/watch?v=CKncw6mIMJQ&t=1022s) delivered at the AppSecUSA conference in 2016. The thesis of my presentation is identical to Professor Green's, but I augment the presentation with my own examples from additional research. That said, definitely check out Professor Green's presentations. He combines technical mastery of the subject with eminently understandable explanations; they really are a treat to watch.

Without further ado, an embedded version of the presentation is inlined below. Alternatively, the presentation is also publicly available at [this link](https://slides.com/turingcompl33t/deck/fullscreen).

<iframe src="https://slides.com/turingcompl33t/deck/embed?style=light" width="576" height="420" scrolling="no" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>

### References

The presentation videos I reference above are here:
- [Cryptography is a Systems Problem](https://www.youtube.com/watch?v=uP6np_oKVCk) by Matthew Green.
- [Cryptography in the Age of Heartbleed](https://www.youtube.com/watch?v=CKncw6mIMJQ) by Matthew Green.

Additional resources I consulted while making the presentation are here:
- Practical Cryptographic Systems by Matthew Green
- Chosen Ciphertext Attacks Against Protocols Based on the RSA Encryption Standard PKCS#1 by Daniel Bleichenbacher
- 20 Years of Attacks on the RSA Cryptosystem by Dan Boneh
- The Most Dangerous Code in the World: Validating SSL Certificates in Non-Browser Software by Georgiev et al. 
- Post-Quantum Cryptography by Daniel J. Bernstein
- Serious Cryptography by Jean-Philipoe Aumasson
- Cryptography Engineering by Ferguson, Schneier, Kohno
- RFC 2313 PKCS #1: RSA Encyption, Version 1.5
- RFC 8452 AES-GCM-SIV: Nonce Misuse-Resistant Authenticated Encryption
- RFC 7457 Summarizing Known Attacks on Transport Layer Security (TLS) and Datagram TLS (DTLS)
- RFC 5246 The Transport Layer Security (TLS) Protocol, Version 1.2
