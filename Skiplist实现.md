```
#ifndef SKIPLIST_H_
#define SKIPLIST_H_

#include <stdio.h>
#include <stdlib.h>

#define MAX_LEVEL 12

struct Skiplist {

	struct Node {
		Node(int k, int v, int l) : key(k), value(v) {
			forwards = new Node*[l];
			for (int i = 0; i < l; ++i)
				forwards[i] = nullptr;
		}
		~Node() {
			delete[] forwards;
		}

		int key;
		int value;
		Node **forwards;
	};

	int level;
	Node *header;

	Skiplist() {
		level = 0;
		header = new Node(0, 0, MAX_LEVEL);
	}

	int RandomLevel() {
		int k = 1;
		while (::rand() % 2) {
			k++;
		}

		k = (k < MAX_LEVEL)? k : MAX_LEVEL;
		return k - 1;
	}

	bool Insert(int k, int v) {
		Node *update[MAX_LEVEL];
		Node *p, *q = nullptr;
		p = header;

		// find the position from top
		for (int l = level; l >= 0; --l) {
			while ((q = p->forwards[l]) && q->key < k) {
				p = q;
			}

			update[l] = p;
		}

		if (q && q->key == k) return false;

		int l = RandomLevel();
		// Update skiplist level
		if (l > level) {
			for (int i = level+1; i <= l; ++i) {
				update[i] = header;
			}
			level = l;
		}

		// insert node per level
		q = new Node(k, v, k + 1);
		for (int i = 0; i <= k; ++i) {
			q->forwards[i] = update[i]->forwards[i];
			update[i]->forwards[i] = q;
		}

		return true;
	}

	bool Find(int k, int* v) {
		Node *p, *q = nullptr;
		p = header;

		for (int i = level; i >= 0; --i) {
			while ((q = p->forwards[i]) && q->key <= k) {
				if (q->key == k) {
					*v = q->value;
					return true;
				}
				p = q;
			}
		}

		return false;
	}

	bool Delete(int k) {
		Node *update[MAX_LEVEL];
		Node *p, *q = nullptr;
		p = header;

		// find the position from top
		for (int l = level; l >= 0; --l) {
			while ((q = p->forwards[l]) && q->key < k) {
				p = q;
			}

			update[l] = p;
		}

		if (q == nullptr || q->key != k) {
			return false;
		}

		// delete
		for (int i = 0; i <= level; ++i) {
			if (update[i]->forwards[i] == q) {
				update[i]->forwards[i] = q->forwards[i];
			}
		}

		delete q;

		for (int i = level; i >= 0; --i) {
			if (header->forwards[i] == nullptr)
				level--;
		}
		return true;
	}

	void Print() const {
		Node *p, *q = nullptr;
		for (int i = level; i >= 0; --i) {
			p = header;
			printf("%d: ", i);
			while (p->forwards[i]) {
				printf("%d -> ", p->forwards[i]->key);
				p = p->forwards[i];
			}
			printf("NULL\n");
		}

		printf("\n");
	}
};

#endif // SKIPLIST_H_
```
