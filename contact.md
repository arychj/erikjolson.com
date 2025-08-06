---
layout: page
title: Contact
permalink: /contact/
---

<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.0.0-beta3/dist/css/bootstrap.min.css">
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.8.0/font/bootstrap-icons.css">
<form method="POST" action="https://contact.erikjolson.com/submit">
    <div class="form-floating mb-3">
        <input type="text" class="form-control" id="name" name="name" placeholder="name@example.com" required>
        <label for="name">Your name</label>
    </div>
    <div class="form-floating mb-3">
        <input type="email" class="form-control" id="address" name="address" placeholder="Anon Nemo" required>
        <label for="address">Your address</label>
    </div>
    <div class="form-floating mb-3">
        <input type="text" class="form-control" id="subject" name="subject" placeholder="Subject" required>
        <label for="subject">Subject</label>
    </div>
    <div class="form-floating mb-3">
        <textarea class="form-control" placeholder="Leave a comment here" id="message" name="message" required></textarea>
        <label for="message">What's up?</label>
    </div>

    <button class="btn btn-primary float-end" type="submit">
        Send
        <i class="bi bi-send"></i>
    </button>
</form>

PGP Public Key
0B8A 3CC1 0996 BFC2 18A5 6E23 5055 CB8E 0B4F 6BF4

