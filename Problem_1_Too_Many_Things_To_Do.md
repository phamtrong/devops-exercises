grep '"symbol": "TSLA"' ./transaction-log.txt | grep '"side": "sell"' | grep -o '"order_id": *"[^"]*"' | cut -d '"' -f4 | xargs -I {} curl -s "https://example.com/api/{}" >> ./output.txt
