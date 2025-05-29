project(
  'scikits.umfpack',
  'c',
  version: '0.4.1',
  license: 'BSD-3',
  default_options: [
    'c_std=c99',
  ]
)

fs = import('fs')
py = import('python').find_installation(pure: false)

cc = meson.get_compiler('c')

swig = find_program('swig', required: true, native: true)

# Try to find suitesparse if it is installed in the current python installation
# (i.e. a conda env). We may end up with a problem if the CMake/pkg-config
# version is different from the one installed in the environment.
#
# Even if pkg-config finds the library, we still need to try to find the
# include dir so we can pass it to SWIG, because Meson does not support SWIG.
# see https://github.com/mesonbuild/meson/issues/341.
_conda_try_incdirs = []
_conda_try_libdirs= []
_py_data_path = py.get_path('data', 'UNKNOWN')
if py.has_path('data')
  # Anaconda defaults and conda-forge on Windows
  _conda_try_incdirs += _py_data_path / 'Library' / 'include'
  _conda_try_incdirs += _py_data_path / 'Library' / 'include' / 'suitesparse'

  _conda_try_libdirs += _py_data_path / 'Library' / 'lib'

  # Linux
  _conda_try_incdirs += _py_data_path / 'include'
  _conda_try_incdirs += _py_data_path / 'include' / 'suitesparse'

  _conda_try_libdirs += _py_data_path / 'lib'
endif

# The list of paths here was taken over from numpy.distutils.system_info.py:
_try_incdirs = [
    '/usr/include/suitesparse',
    '/usr/local/include',
    '/opt/include',
    '/opt/local/include/ufsparse',
    '/opt/local/include',
    '/sw/include',
    '/usr/include/suitesparse',
    '/opt/homebrew/include/suitesparse'
]
suitesparse_incdirs = []
umfpack_header_dirs = []
foreach _dir : _conda_try_incdirs + _try_incdirs
  if fs.exists(_dir)
    suitesparse_incdirs += _dir
    umfpack_header_dirs += include_directories(_dir)
  endif
endforeach

# It may be worth making umfpack_libdirs explicitly override the rest of the
# library search, so that a given suitesparse installation can be specified.
umfpack_libdirs = []
umfpack_libdir = meson.get_external_property('umfpack-libdir', 'UNKNOWN')
umfpack_includedir = meson.get_external_property('umfpack-includedir', 'UNKNOWN')
if umfpack_libdir != 'UNKNOWN'
    umfpack_libdirs += umfpack_libdir
endif
if umfpack_includedir != 'UNKNOWN'
    suitesparse_incdirs += umfpack_includedir
    umfpack_header_dirs += include_directories(umfpack_includedir)
endif

foreach _dir : _conda_try_libdirs
  if fs.exists(_dir)
    umfpack_libdirs += _dir
  endif
endforeach

# Find UMFPACK library.
# Note that Suitesparse packaging is messy; the headers may be found under
# either `<prefix>/include/` or <prefix>/suitesparse/include`. Conda-forge does
# the former, Ubuntu the latter. Usage in this package is `#include <umfpack.h>`,
# so we need to add the suitesparse/umfpack.h to the include directories.
#
# Suitesparse recently started supporting pkg-config (see
# https://github.com/DrTimothyAldenDavis/SuiteSparse/pull/356) and CMake, so try
# with `dependency()` first. Packagers that run into an issue can add a path
# below, but it'd be better if they packaged the Suitesparse .pc files.
umfpack_dep = dependency('UMFPACK', required: false)
if not umfpack_dep.found()
  # Using pkg-config or CMake didn't work, so try the "manual" way.
  umfpack_lib = cc.find_library('umfpack',
      required : true,
      dirs: umfpack_libdirs,
      header_include_directories: umfpack_header_dirs,
      has_headers : ['umfpack.h']
  )
  umfpack_dep = declare_dependency(
    dependencies: umfpack_lib,
    include_directories: umfpack_header_dirs,
  )
  # Check whether we have the single-header version. May break if multiple
  # versions installed in different places.
  single_header = not cc.has_header(
    'umfpack_solve.h',
    dependencies: umfpack_dep,
    include_directories: umfpack_header_dirs
  )
else
  # pkg-config .pc files were only added to recent versions of suitesparse
  single_header = true
endif


swig_defines =[]
if single_header
  message('Using single-header UMFPACK')
  swig_defines += ['-DSK_UMFPACK_SINGLE_HEADER']
else
  message('Multiple-header UMFPACK found')
endif

blas_deps = []

# The single-header version does not need to be linked against BLAS
# This may have been from version 6.1.0 of UMFPACK (Jan 2023) as the changelog
# says BLAS symbols no longer exposed to user applications.
# Single header and int64_t is used from v6.0.0 (Nov 2022)
if not single_header
  if host_machine.system() == 'darwin'
    blas_deps = [dependency('Accelerate')]
  else
    blas_deps = [dependency('openblas', required : false)]  # try with pkg-config first
    if not blas_deps[0].found()
      blas_deps = [cc.find_library('openblas', required : false)]
    endif
  endif
  if not blas_deps[0].found()
      blas_deps = [cc.find_library('blas')]
      cblas_dep = cc.find_library('cblas', required : false)
      if cblas_dep.found()
          blas_deps += cblas_dep
      endif
  endif
endif

subdir('scikits')
