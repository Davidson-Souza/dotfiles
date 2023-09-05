pkgname=rustreexo-git
pkgver=1.0
pkgrel=0
maintainer="Davidson-Souza <davidson.lucas.souza@outlook.com>"
arch=('any')
url='https://github.com/Davidson-Souza/rustreexo#branch=python-bindings'
license=('MIT')
makedepends=('cargo')
provides=("librustreexo.so.1" "librustreexo.a.1")

source=('git+https://github.com/Davidson-Souza/rustreexo#branch=python-bindings')
sha256sums=('SKIP')

build() {
    cd "${srcdir}/rustreexo/c-rustreexo/"
    cargo build --release
}

package() {
    cd "${srcdir}/rustreexo/"
                                                
    install -Dm 755 "target/release/librustreexo.so" "${pkgdir}/usr/lib/librustreexo.so"

    install -Dm 755 "target/release/librustreexo.a" "${pkgdir}/usr/lib/librustreexo.a"
    
    for file in $(ls "c-rustreexo/include/"); do
        install -Dm 644 "c-rustreexo/include/$file" "${pkgdir}/usr/include/${file}"
    done
    install -Dm 644 LICENSE "${pkgdir}/usr/share/licenses/librustreexo/LICENSE"
}
