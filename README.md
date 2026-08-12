# Big O notatsiyasi - amaliy misollar

# O(1) - Doimiy vaqt: ro'yxat elementiga indeks orqali murojaat
def get_first_element(arr):
    # Har doim bitta amal, ro'yxat hajmidan qat'i nazar
    return arr[0]


# O(n) - Chiziqli vaqt: ro'yxat bo'ylab bir marta aylanish
def find_max(arr):
    max_val = arr[0]
    for i in range(1, len(arr)):
        # Har bir elementni bitta marta tekshiramiz
        if arr[i] > max_val:
            max_val = arr[i]
    return max_val


# O(n^2) - Kvadratik vaqt: ichma-ich ikkita loop
def has_duplicates(arr):
    for i in range(len(arr)):
        for j in range(i + 1, len(arr)):
            # Har bir juftlikni solishtiramiz - taxminan n * n amal
            if arr[i] == arr[j]:
                return True
    return False


# O(log n) - Logarifmik vaqt: binary search
def binary_search(sorted_arr, target):
    left = 0
    right = len(sorted_arr) - 1

    while left <= right:
        # Har safar qidiruv maydonini yarmiga qisqartiramiz
        mid = (left + right) // 2

        if sorted_arr[mid] == target:
            return mid
        elif sorted_arr[mid] < target:
            left = mid + 1
        else:
            right = mid - 1
    return -1


# O(n log n) - samarali tartiblash uchun Python'ning o'rnatilgan sorted()
def sort_numbers(arr):
    # Python Timsort algoritmidan foydalanadi - o'rtacha O(n log n)
    return sorted(arr)


# Amaliyot uchun test
numbers = [5, 3, 8, 1, 9, 2, 7]
print("Max element (O(n)):", find_max(numbers))
print("Duplicates bormi (O(n^2)):", has_duplicates(numbers))

sorted_numbers = sort_numbers(numbers)
print("Tartiblangan (O(n log n)):", sorted_numbers)
print("Binary search natijasi (O(log n)):", binary_search(sorted_numbers, 8))

# Har bir funksiya uchun murakkablikni izohlash muhim,
# chunki bu real loyihalarda performance muammolarini
# oldindan aniqlash imkonini beradi.
